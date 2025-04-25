# **Enhance MQTT 5.0 with Batch Publishing**

## **1\. Introduction**

### **1.1 Purpose**

This proposal outlines a design for sending multiple logical application messages (sub-messages) within the payload of a single MQTT 5.0 PUBLISH packet. The design leverages MQTT 5.0 User Properties to signal the batch format and size, aiming to improve throughput and efficiency compared to sending numerous small, individual messages.

### **1.2 Motivation**

Transmitting many small, individual messages via MQTT can lead to:
- Increased network overhead due to repeated transmission of packet headers and topic names.
- Higher CPU utilization for packet processing and message routing.
- Reduced throughput and efficiency compared to sending larger payloads.
- Increased the latency due to multiple round trips.

This batching mechanism addresses these issues by consolidating messages destined for the same topic and sharing the same PUBLISH packet attributes (QoS, Retain flag, etc.) into one MQTT transmission, while remaining compliant with the MQTT 5.0 standard.

## **2\. Design**

### **2.1 Packet Structure**

The batch is contained within the payload of a standard MQTT 5.0 PUBLISH packet. Specific User Properties signal the batch nature and format.

MQTT 5.0 PUBLISH Packet  
├── Fixed Header (Type=PUBLISH, QoS, DUP, RETAIN flags)  
├── Remaining Length (Variable Byte Integer)  
├── Variable Header  
│   ├── Topic Name (UTF-8 string)  
│   ├── Packet Identifier (2 bytes, present only if QoS \> 0\)  
│   └── Properties  
│       ├── User Property: "batch-format" \= "v2"  \[2\]  
│       ├── User Property: "batch-size" \= "N"   \[2\]  
│       └──... (Other standard MQTT 5.0 properties like Message Expiry, Content-Type, etc.)  
└── Payload (Batch Payload)  
    ├── Length1 (Variable Byte Integer) \[2, 3\]  
    ├── SubPayload1 (variable bytes)  
    ├── Length2 (Variable Byte Integer) \[2, 3\]  
    ├── SubPayload2 (variable bytes)  
    └──... (Up to N sub-messages)

### **2.2 Property Definitions**

* **batch-format (User Property):** MUST be present. A UTF-8 string identifying the version of this batch payload specification (e.g., "v2"). Subscribers use this to select the correct parsing logic.2  
* **batch-size (User Property):** MUST be present. A UTF-8 string representing a positive integer (N) indicating the exact number of logical sub-messages contained within the Batch Payload.2

### **2.3 Payload Format (Batch Payload)**

The Batch Payload consists of N concatenated sub-messages, where N is the integer value specified by the batch-size User Property. Each sub-message MUST be structured as:

* **Length:** A Variable Byte Integer (VBI) as defined in the MQTT 5.0 specification 2, indicating the number of bytes in the subsequent SubPayload. The VBI encoding allows lengths from 0 up to 268,435,455 bytes using 1 to 4 bytes for the length prefix.2  
* **SubPayload:** The actual application data bytes for the logical sub-message. The format of this data is application-specific. Zero-length SubPayloads (Length \= 0\) MAY be permitted based on application policy.

### **2.4 Example (Conceptual)**

For a batch containing two sub-messages, "Msg1" (4 bytes) and "LongerMsg2" (10 bytes):  
Properties:  
User Property: batch-format \= v2  
User Property: batch-size \= 2  
Payload:  
0x04 Msg1 0x0A LongerMsg2  
(Note: 0x04 and 0x0A represent the single-byte VBI encoding for lengths 4 and 10 respectively) 2

## **3\. Implementation Requirements**

### **3.1 Publisher Requirements**

* MUST set the batch-format User Property to a supported version string (e.g., "v2").  
* MUST set the batch-size User Property to the correct positive integer count of sub-messages in the batch, encoded as a UTF-8 string.  
* MUST format the Batch Payload according to Section 2.3, using VBI for length prefixes.2  
* MUST ensure the total size of the generated MQTT PUBLISH packet (including headers, properties, and Batch Payload) does not exceed the Maximum Packet Size limit communicated by the broker in the CONNACK packet, if any.2  
* SHOULD implement and respect configurable upper limits for batch-size and the total byte size of the Batch Payload (see Section 3.3).  
* SHOULD validate sub-payload sizes before sending.

### **3.2 Subscriber Requirements**

* MUST check for the presence and value of the batch-format User Property to ensure compatibility.  
* MUST check for the presence of the batch-size User Property and parse its integer value.  
* MUST parse the Batch Payload according to Section 2.3, correctly decoding VBI lengths.2  
* MUST perform rigorous validation checks as defined in Section 4 (Error Handling) and Section 7 (Security Considerations).  
* MUST handle malformed batches according to the error handling strategies defined in Section 4\.  
* SHOULD implement and respect configurable upper limits for batch-size and the total byte size of the Batch Payload (see Section 3.3).

### **3.3 Operational Limits**

* Both Publishers and Subscribers SHOULD implement configurable upper limits to prevent resource exhaustion.6 Recommended defaults:  
  * max-batch-size (sub-message count): 100  
  * max-total-batch-payload-size (bytes): 65536  
* Implementers MUST adjust these limits based on system constraints, network conditions (e.g., MTU 9), and broker capabilities.  
* Subscribers MUST reject batches exceeding their configured limits (log appropriate error, e.g., BATCH\_SIZE\_LIMIT\_EXCEEDED).

## **4\. Error Handling (Subscriber)**

Subscribers MUST handle malformed batches robustly to ensure stability and prevent crashes or incorrect processing.

### **4.1 Malformed Batch Scenarios**

Subscribers MUST detect the following error conditions during parsing:

1. **Missing/Invalid Properties:** batch-format or batch-size User Property missing, or batch-size is not a positive integer string.  
2. **Unsupported Format:** batch-format value indicates a version not supported by the subscriber.  
3. **Invalid VBI Length:** VBI sequence for a sub-payload length is malformed (e.g., \> 4 bytes) or decodes to an invalid value (e.g., negative, or zero if disallowed by policy).2  
4. **Length Exceeds Payload Boundary:** A decoded VBI length value is greater than the number of bytes remaining in the Batch Payload buffer.  
5. **Incomplete Payload Data:** The Batch Payload ends prematurely while reading a VBI length or its corresponding sub-payload data.  
6. **Count Mismatch:** The number of sub-messages successfully parsed does not match the value specified in batch-size (either fewer messages found, or trailing data exists after the Nth message).

### **4.2 Handling Strategies**

* **Mandatory Discard:** For errors 1, 2, 3, 4, or 5, the subscriber MUST discard the entire Batch Payload and MUST NOT process any sub-messages. An error MUST be logged detailing the topic, properties, and a specific reason code (e.g., MALFORMED\_BATCH\_MISSING\_PROPERTY, MALFORMED\_BATCH\_INVALID\_LENGTH, MALFORMED\_BATCH\_INCOMPLETE\_PAYLOAD).  
* **Configurable Count Mismatch (Error 6):**  
  * **Default:** Discard the entire batch and log an error (e.g., MALFORMED\_BATCH\_COUNT\_MISMATCH).  
  * **Optional Partial Processing:** Allow configuration to process the valid sub-messages parsed *before* the mismatch was detected. If enabled and used, MUST log a warning indicating partial processing, messages processed vs. expected count, and the reason (fewer found / trailing data).  
* **Logging:** Error/warning logs MUST include sufficient context (topic, client ID if available, batch properties) and specific error indicators.

## **5\. Interaction with MQTT 5.0 Features**

Core MQTT 5.0 features apply to the PUBLISH packet containing the batch as a single unit.2

* **Retain Flag (RETAIN=1):** If set, the broker MUST store the *entire* PUBLISH packet (with the full Batch Payload and all properties) as the single retained message for the topic, replacing any previous one.2 New subscribers receive this full batch message. Clearing requires publishing a zero-byte payload with RETAIN=1.2  
* **QoS Levels (0, 1, 2):** The QoS level applies to the delivery assurance of the *entire batch packet* between network hops.2  
  * QoS 1 (PUBACK) and QoS 2 (PUBREC/PUBREL/PUBCOMP) acknowledgments confirm delivery of the *batch packet*, identified by its Packet Identifier.2  
  * **Crucially, MQTT QoS does NOT provide application-level atomicity for the sub-messages within the batch.** A subscriber might acknowledge receipt (PUBACK/PUBREC) but fail during subsequent parsing/processing of the Batch Payload. Applications requiring all-or-nothing processing of sub-messages MUST implement their own end-to-end acknowledgment mechanism.  
* **Message Expiry Interval:** If set in the PUBLISH properties, the interval applies to the *entire Application Message* (the batch).2 The broker MUST discard the message for undelivered subscribers if the interval expires before delivery starts.2 The interval is reduced by the time spent waiting on the broker when forwarded.2

## **6\. Comparison with Alternatives**

While this specification uses User Properties and a custom binary format, alternatives exist 2:

* **JSON Array Payload:** Uses Payload Format Indicator=1, Content Type=application/json. Human-readable, uses standard libraries, but higher overhead and parsing cost.20  
* **Protobuf/MessagePack Payload:** Uses Payload Format Indicator=0, appropriate Content Type. More efficient binary formats, faster parsing than JSON, but require specific libraries/schemas (Protobuf) and are not human-readable.2

The User Property approach is chosen for its flexibility in defining the binary format (allowing potential for minimal overhead), clear signaling of intent via standard MQTT 5.0 properties 2, and decoupling metadata from the payload. This requires custom parsing logic but offers control over the format.

## **7\. Security Considerations**

Implementing this batching mechanism requires careful attention to security.

* **Authentication & Authorization (Mandatory):**  
  * Clients publishing or subscribing to batch topics MUST be authenticated by the broker (e.g., username/password, client certificates).27  
  * Brokers MUST use Access Control Lists (ACLs) to enforce that only authorized clients can publish/subscribe to specific batch topics, following the principle of least privilege.30  
* **Transport Security (Mandatory):**  
  * Connections MUST use TLS (typically port 8883\) to encrypt the entire MQTT communication (headers, properties, payload) and protect against eavesdropping and tampering.27 Client and server certificate validation MUST be performed.33  
* **Payload Validation (Mandatory):**  
  * Subscribers MUST perform rigorous structural validation of the Batch Payload as defined in Section 4 (Error Handling) before attempting to process sub-payloads.27 Parsers MUST fail safely on error.  
  * Applications SHOULD perform content validation on each individual SubPayload based on application-specific logic (e.g., data type, range checks) after successful structural parsing.27  
* **Denial of Service (DoS) Mitigation (Mandatory):**  
  * Enforce configurable limits on batch-size and max-total-batch-payload-size (Section 3.3) on both publisher and subscriber to prevent resource exhaustion attacks.35  
  * Brokers SHOULD implement rate limiting (connections/sec, messages/sec per client) to mitigate high-frequency and flooding attacks.24  
* **Optional Payload Security:**  
  * For end-to-end security where the broker is untrusted, consider encrypting the *entire* Batch Payload at the application level (e.g., AES).44  
  * Consider signing or using HMAC on the *entire* Batch Payload to ensure integrity and authenticity from the publisher.30 Note: Application-level crypto adds overhead.44

## **8\. Future Considerations**

* **Versioning:** New batch formats can be introduced by defining new values for the batch-format User Property (e.g., "v3"). Subscribers would need to support multiple versions.  
* **Compression:** To reduce bandwidth, the *entire* Batch Payload MAY be compressed (e.g., using gzip, Zstandard). Compression SHOULD be applied *after* constructing the VBI-prefixed batch payload. The use of compression SHOULD be signaled using the standard MQTT 5.0 Content Type property (e.g., application/vnd.mqtt-batch.v2+gzip) rather than a User Property.46  
* **Application-Level Priority:** Priority among sub-messages within a batch is an application-level concern. It would require adding priority information within the SubPayload format and implementing corresponding logic in the subscriber. It is NOT handled by MQTT itself.

## **9\. Conclusion**

This specification provides a framework for batching multiple logical messages within a single MQTT 5.0 PUBLISH packet using User Properties and a VBI-based payload format. By adhering to the implementation requirements, error handling procedures, and security considerations outlined herein, applications can leverage this mechanism to improve efficiency while maintaining protocol compliance and robustness.

#### **Works cited**

1. Best Practices For Mqtt In Home Automation | Restackio, accessed April 24, 2025, [https://www.restack.io/p/open-source-home-automation-ai-frameworks-answer-mqtt-best-practices-cat-ai](https://www.restack.io/p/open-source-home-automation-ai-frameworks-answer-mqtt-best-practices-cat-ai)  
2. mqtt-v5.0.html \- MQTT Version 5.0 \- OASIS Open, accessed April 24, 2025, [https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html)  
3. MQTT Control Packets: A Beginner's Guide | EMQ \- EMQX, accessed April 24, 2025, [https://www.emqx.com/en/blog/introduction-to-mqtt-control-packets](https://www.emqx.com/en/blog/introduction-to-mqtt-control-packets)  
4. User Properties \- MQTT 5.0 new features | EMQ \- EMQX, accessed April 24, 2025, [https://www.emqx.com/en/blog/mqtt5-user-properties](https://www.emqx.com/en/blog/mqtt5-user-properties)  
5. MQTT V3.1 Protocol Specification \- IBM, accessed April 24, 2025, [https://public.dhe.ibm.com/software/dw/webservices/ws-mqtt/mqtt-v3r1.html](https://public.dhe.ibm.com/software/dw/webservices/ws-mqtt/mqtt-v3r1.html)  
6. UA Part 2: Security \- 4.3.2 Denial of Service, accessed April 24, 2025, [https://reference.opcfoundation.org/Core/Part2/v104/docs/4.3.2](https://reference.opcfoundation.org/Core/Part2/v104/docs/4.3.2)  
7. Denial of service attack detection through machine learning for the IoT \- ResearchGate, accessed April 24, 2025, [https://www.researchgate.net/publication/342140035\_Denial\_of\_service\_attack\_detection\_through\_machine\_learning\_for\_the\_IoT](https://www.researchgate.net/publication/342140035_Denial_of_service_attack_detection_through_machine_learning_for_the_IoT)  
8. Azure IoT Hub MQTT 5 support (preview) \- Learn Microsoft, accessed April 24, 2025, [https://learn.microsoft.com/en-us/azure/iot/iot-mqtt-5-preview](https://learn.microsoft.com/en-us/azure/iot/iot-mqtt-5-preview)  
9. What Is MTU Size and How Do You Calculate It? \- Pure Storage Blog, accessed April 24, 2025, [https://blog.purestorage.com/purely-educational/what-is-mtu-size-and-how-do-you-calculate-it/](https://blog.purestorage.com/purely-educational/what-is-mtu-size-and-how-do-you-calculate-it/)  
10. Maximum transmission unit \- Wikipedia, accessed April 24, 2025, [https://en.wikipedia.org/wiki/Maximum\_transmission\_unit](https://en.wikipedia.org/wiki/Maximum_transmission_unit)  
11. docs.eseye.com, accessed April 24, 2025, [https://docs.eseye.com/Content/Connectivity/MTU.htm\#:\~:text=For%20Ethernet%2Dbased%20networks%20(including,radio%20links%20to%20transmit%20data.](https://docs.eseye.com/Content/Connectivity/MTU.htm#:~:text=For%20Ethernet%2Dbased%20networks%20\(including,radio%20links%20to%20transmit%20data.)  
12. What are Retained Messages in MQTT? – MQTT Essentials: Part 8 \- HiveMQ, accessed April 24, 2025, [https://www.hivemq.com/blog/mqtt-essentials-part-8-retained-messages/](https://www.hivemq.com/blog/mqtt-essentials-part-8-retained-messages/)  
13. MQTT Retain Flag and Potential Problems \- GitHub Gist, accessed April 24, 2025, [https://gist.github.com/machinekoder/3ba0e8a7172c0804bc3e68e25ec49bed](https://gist.github.com/machinekoder/3ba0e8a7172c0804bc3e68e25ec49bed)  
14. MQTT · ActiveMQ Artemis Documentation, accessed April 24, 2025, [https://activemq.apache.org/components/artemis/documentation/2.29.0/mqtt.html](https://activemq.apache.org/components/artemis/documentation/2.29.0/mqtt.html)  
15. What is MQTT Quality of Service (QoS) 0,1, & 2? – MQTT Essentials: Part 6 \- HiveMQ, accessed April 24, 2025, [https://www.hivemq.com/blog/mqtt-essentials-part-6-mqtt-quality-of-service-levels/](https://www.hivemq.com/blog/mqtt-essentials-part-6-mqtt-quality-of-service-levels/)  
16. Introduction to MQTT Message Expiry Interval | MQTT 5 Features | EMQ \- EMQX, accessed April 24, 2025, [https://www.emqx.com/en/blog/mqtt-message-expiry-interval](https://www.emqx.com/en/blog/mqtt-message-expiry-interval)  
17. MQTT Session Expiry and Message Expiry Intervals – MQTT 5 Essentials Part 4 \- HiveMQ, accessed April 24, 2025, [https://www.hivemq.com/blog/mqtt5-essentials-part4-session-and-message-expiry/](https://www.hivemq.com/blog/mqtt5-essentials-part4-session-and-message-expiry/)  
18. Introduction to MQTT 5 and comparison with version 3.1 \- Bevywise, accessed April 24, 2025, [https://www.bevywise.com/blog/introduction-to-mqtt-5/](https://www.bevywise.com/blog/introduction-to-mqtt-5/)  
19. MQTT v5.0 versus v3.1.1 \- wolfSSL, accessed April 24, 2025, [https://www.wolfssl.com/mqtt-v5-0-versus-v3-1-1/](https://www.wolfssl.com/mqtt-v5-0-versus-v3-1-1/)  
20. Best way to pack/unpack multiple items in a JSON MQTT payload ..., accessed April 24, 2025, [https://community.openhab.org/t/best-way-to-pack-unpack-multiple-items-in-a-json-mqtt-payload/158593](https://community.openhab.org/t/best-way-to-pack-unpack-multiple-items-in-a-json-mqtt-payload/158593)  
21. protobuf \+ MQTT is yummy fast goodness | IoT, Cognition and Cars ..., accessed April 24, 2025, [https://mobilebit.wordpress.com/2013/09/25/protobuf-mqtt-yummy-fast/](https://mobilebit.wordpress.com/2013/09/25/protobuf-mqtt-yummy-fast/)  
22. MessagePack: It's like JSON, but fast and small. | Hacker News, accessed April 24, 2025, [https://news.ycombinator.com/item?id=42663047](https://news.ycombinator.com/item?id=42663047)  
23. deserialization \- Performant Entity Serialization: BSON vs ..., accessed April 24, 2025, [https://stackoverflow.com/questions/6355497/performant-entity-serialization-bson-vs-messagepack-vs-json](https://stackoverflow.com/questions/6355497/performant-entity-serialization-bson-vs-messagepack-vs-json)  
24. Exploring Alternatives: Are There Better Options Than JSON? \- DEV Community, accessed April 24, 2025, [https://dev.to/gin\_mitch/exploring-alternatives-are-there-better-options-than-json-3mc5](https://dev.to/gin_mitch/exploring-alternatives-are-there-better-options-than-json-3mc5)  
25. MQTT 5 Essentials Part 05 \- User Properties \- YouTube, accessed April 24, 2025, [https://www.youtube.com/watch?v=yBf33-QTN\_Q](https://www.youtube.com/watch?v=yBf33-QTN_Q)  
26. MQTT5 User Properties with Mosquitto Bridge \- Stack Overflow, accessed April 24, 2025, [https://stackoverflow.com/questions/75283630/mqtt5-user-properties-with-mosquitto-bridge](https://stackoverflow.com/questions/75283630/mqtt5-user-properties-with-mosquitto-bridge)  
27. Protecting Your IoT Infrastructure: Essential MQTT Security Practices \- WebbyLab, accessed April 24, 2025, [https://webbylab.com/blog/protecting-your-iot-infrastructure-essential-mqtt-security-practices/](https://webbylab.com/blog/protecting-your-iot-infrastructure-essential-mqtt-security-practices/)  
28. Securing MQTT: Best Practices for a Robust IoT Ecosystem \- Cirrus Link Solutions, accessed April 24, 2025, [https://cirrus-link.com/wp-content/uploads/2024/01/Securing-MQTT-Best-Practices-for-a-Robust-IoT-Ecosystem.pdf](https://cirrus-link.com/wp-content/uploads/2024/01/Securing-MQTT-Best-Practices-for-a-Robust-IoT-Ecosystem.pdf)  
29. Best security practices in MQTT protocol \- Tech Blogs \- 42Gears, accessed April 24, 2025, [https://techblogs.42gears.com/best-security-practices-in-mqtt-protocol/](https://techblogs.42gears.com/best-security-practices-in-mqtt-protocol/)  
30. MQTT Security: Best Practices for Secure IoT Communications | MQTT.pro, accessed April 24, 2025, [https://mqtt.pro/mqtt-security/](https://mqtt.pro/mqtt-security/)  
31. How Hackers Can Easily Penetrate Your MQTT Solution \- Real Time Logic, accessed April 24, 2025, [https://realtimelogic.com/articles/How-Hackers-Can-Easily-Penetrate-Your-MQTT-Solution](https://realtimelogic.com/articles/How-Hackers-Can-Easily-Penetrate-Your-MQTT-Solution)  
32. mqtt \- Mosquitto TLS Security \- Can the message payload be read? \- Stack Overflow, accessed April 24, 2025, [https://stackoverflow.com/questions/75249265/mosquitto-tls-security-can-the-message-payload-be-read](https://stackoverflow.com/questions/75249265/mosquitto-tls-security-can-the-message-payload-be-read)  
33. Security with MQTT Protocol \- 5GCroCo, accessed April 24, 2025, [https://5gcroco.eu/images/templates/rsvario/images/5GCroCo\_MQTT\_webseminar.pdf](https://5gcroco.eu/images/templates/rsvario/images/5GCroCo_MQTT_webseminar.pdf)  
34. What are the best practices to secure MQTT \- Reddit, accessed April 24, 2025, [https://www.reddit.com/r/MQTT/comments/9ocfny/what\_are\_the\_best\_practices\_to\_secure\_mqtt/](https://www.reddit.com/r/MQTT/comments/9ocfny/what_are_the_best_practices_to_secure_mqtt/)  
35. AI techniques for automated penetration testing in MQTT networks: a literature investigation, accessed April 24, 2025, [https://www.tandfonline.com/doi/full/10.1080/1206212X.2024.2443504?af=R](https://www.tandfonline.com/doi/full/10.1080/1206212X.2024.2443504?af=R)  
36. IoT-MQTT based denial of service attack modelling and detection \- CORE, accessed April 24, 2025, [https://core.ac.uk/download/323493951.pdf](https://core.ac.uk/download/323493951.pdf)  
37. Full article: Denial of service attack detection through machine learning for the IoT, accessed April 24, 2025, [https://www.tandfonline.com/doi/full/10.1080/24751839.2020.1767484](https://www.tandfonline.com/doi/full/10.1080/24751839.2020.1767484)  
38. Detection of TCP and MQTT-Based DoS/DDoS Attacks on MUD IoT Networks \- MDPI, accessed April 24, 2025, [https://www.mdpi.com/2079-9292/14/8/1653](https://www.mdpi.com/2079-9292/14/8/1653)  
39. Software-Defined-Networking-Based One-versus-Rest Strategy for Detecting and Mitigating Distributed Denial-of-Service Attacks in Smart Home Internet of Things Devices \- PMC \- PubMed Central, accessed April 24, 2025, [https://pmc.ncbi.nlm.nih.gov/articles/PMC11314767/](https://pmc.ncbi.nlm.nih.gov/articles/PMC11314767/)  
40. What are MQTT User Properties? – MQTT 5 Essentials Part 6 \- HiveMQ, accessed April 24, 2025, [https://www.hivemq.com/blog/mqtt5-essentials-part6-user-properties/](https://www.hivemq.com/blog/mqtt5-essentials-part6-user-properties/)  
41. Machine leaning-based DoS attacks detection for MQTT sensor networks \- POLITesi, accessed April 24, 2025, [https://www.politesi.polimi.it/bitstream/10589/178090/1/2021\_7\_Ghannadrad.pdf](https://www.politesi.polimi.it/bitstream/10589/178090/1/2021_7_Ghannadrad.pdf)  
42. INTELLIGENT SYSTEMS AND APPLICATIONS IN ENGINEERING Feasibility Review of DDoS Attack Mitigation on CoAP for IoT Networks, accessed April 24, 2025, [https://ijisae.org/index.php/IJISAE/article/download/5648/4387/10721](https://ijisae.org/index.php/IJISAE/article/download/5648/4387/10721)  
43. UA Part 14: PubSub \- 7.3.5 MQTT, accessed April 24, 2025, [https://reference.opcfoundation.org/Core/Part14/v105/docs/7.3.5](https://reference.opcfoundation.org/Core/Part14/v105/docs/7.3.5)  
44. MQTT Security Fundamentals \- Payload Encryption \- HiveMQ, accessed April 24, 2025, [https://www.hivemq.com/blog/mqtt-security-fundamentals-payload-encryption/](https://www.hivemq.com/blog/mqtt-security-fundamentals-payload-encryption/)  
45. MQTT Message Data Integrity \- MQTT Security Fundamentals \- HiveMQ, accessed April 24, 2025, [https://www.hivemq.com/blog/mqtt-security-fundamentals-mqtt-message-data-integrity/](https://www.hivemq.com/blog/mqtt-security-fundamentals-mqtt-message-data-integrity/)  
46. Introduction to MQTT Payload Format Indicator and Content Type | MQTT 5 Features \- EMQX, accessed April 24, 2025, [https://www.emqx.com/en/blog/mqtt5-new-features-payload-format-indicator-and-content-type](https://www.emqx.com/en/blog/mqtt5-new-features-payload-format-indicator-and-content-type)  
47. MQTT Payload Format Description and Content Type – MQTT 5 Essentials Part 8 \- HiveMQ, accessed April 24, 2025, [https://www.hivemq.com/blog/mqtt5-essentials-part8-payload-format-description/](https://www.hivemq.com/blog/mqtt5-essentials-part8-payload-format-description/)  
48. Send compressed JSON with MQTT.js \- Stack Overflow, accessed April 24, 2025, [https://stackoverflow.com/questions/40047723/send-compressed-json-with-mqtt-js](https://stackoverflow.com/questions/40047723/send-compressed-json-with-mqtt-js)  
49. MQTT data compression \- Stack Overflow, accessed April 24, 2025, [https://stackoverflow.com/questions/61247631/mqtt-data-compression](https://stackoverflow.com/questions/61247631/mqtt-data-compression)  
50. 0005800: For the MQTT PubSub encoding, specify a gzipped version of the body/payload for huge cost savings \- MantisBT, accessed April 24, 2025, [https://mantis.opcfoundation.org/view.php?id=5800](https://mantis.opcfoundation.org/view.php?id=5800)  
51. Using MQTT \- Solace Docs, accessed April 24, 2025, [https://docs.solace.com/API/MQTT/Using-MQTT.htm](https://docs.solace.com/API/MQTT/Using-MQTT.htm)  
52. Using MQTT | Documentation \- Docs \- DiffusionData, accessed April 24, 2025, [https://docs.diffusiondata.com/docs/latest/manual/html/administratorguide/mqtt/using\_mqtt.html](https://docs.diffusiondata.com/docs/latest/manual/html/administratorguide/mqtt/using_mqtt.html)