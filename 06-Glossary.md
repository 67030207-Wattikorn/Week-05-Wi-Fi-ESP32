# 06. อภิธานศัพท์และการจำแนกโครงสร้าง (Glossary & Architecture Taxonomy)

เอกสารรวบรวมคำศัพท์ทางเทคนิค (Technical Terms) นิยาม และตัวย่อสำคัญที่เกี่ยวข้องกับ **สถาปัตยกรรมเครือข่าย Wi-Fi (IEEE 802.11)** และการทำงานบนไมโครคอนโทรลเลอร์ **ESP32** 

## 6.1 คำศัพท์เกี่ยวกับบทบาทและองค์ประกอบในเครือข่าย (Network Roles & Identifiers)

| คำศัพท์ / ตัวย่อ | คำเต็ม                             | มิติประเภท           | คำอธิบายและความหมาย                                                                                                               |
| :--------------- | :--------------------------------- | :------------------- | :-------------------------------------------------------------------------------------------------------------------------------- |
| **AP**           | Access Point                       | Hardware / Role      | อุปกรณ์จุดเชื่อมต่อเครือข่ายไร้สาย (เช่น Router หรือ Hotspot) ทำหน้าที่กระจายสัญญาณและจัดการ Client                               |
| **STA**          | Station                            | Hardware / Role      | อุปกรณ์ลูกข่าย Wi-Fi ที่ขอเชื่อมต่อเข้ากับ AP (ในที่นี้คือไมโครคอนโทรลเลอร์ **ESP32**, คอมพิวเตอร์, หรือสมาร์ตโฟน)                |
| **SSID**         | Service Set Identifier             | Data / Identifier    | ชื่อของเครือข่าย Wi-Fi (เช่น `"MyHome_2.4G"`) ใช้สำหรับการระบุเครือข่ายที่ต้องการเชื่อมต่อ                                        |
| **BSSID**        | Basic Service Set Identifier       | Data / Identifier    | หมายเลข MAC Address ของอินเทอร์เฟซวิทยุบน AP ตัวที่ให้บริการ (ใช้แยกแยะเมื่อมีหลาย AP ใช้ SSID เดียวกัน)                          |
| **MAC Address**  | Media Access Control Address       | Data / Identifier    | หมายเลขระบุตัวตนประจำกายภาพของอุปกรณ์ H/W เครือข่าย (ความยาว 48 บิต / 6 ไบต์) มีความซ้ำกันไม่ได้ทั่วโลก                           |
| **RSSI**         | Received Signal Strength Indicator | Data / Signal Metric | ดัชนีวัดระดับความแรงของสัญญาณวิทยุที่รับได้ มีหน่วยเป็น **dBm** (ยิ่งเข้าใกล้ 0 แสดงว่าสัญญาณยิ่งแรง เช่น -40 dBm ดีกว่า -85 dBm) |
| **AID**          | Association ID                     | Data / Identifier    | หมายเลขประจำตัวลูกข่ายที่ AP จัดสรรให้แก่ STA เมื่อขั้นตอน Association สำเร็จ                                                     |

---

## 6.2 คำศัพท์เกี่ยวกับเฟสและการรับส่งแพ็กเกจ (Phases & Frame Types)

| คำศัพท์ / ตัวย่อ             | คำเต็ม                     | มิติประเภท        | คำอธิบายและความหมาย                                                                                                |
| :--------------------------- | :------------------------- | :---------------- | :----------------------------------------------------------------------------------------------------------------- |
| **Active Scanning**          | Active Scanning            | Mechanism / Phase | การสแกนหา Wi-Fi แบบรุก โดย ESP32 ส่ง Probe Request Frame ออกไปค้นหา AP ในแต่ละ Channel                             |
| **Passive Scanning**         | Passive Scanning           | Mechanism / Phase | การสแกนหา Wi-Fi แบบตั้งรับ โดย ESP32 คอยรับฟัง Beacon Frame ที่ AP กระจายออกมาโดยไม่ส่งแพ็กเกจใดๆ                  |
| **Beacon Frame**             | Beacon Frame               | Data / Frame      | แพ็กเกจที่ AP ส่งกระจายออกมาในอากาศเป็นระยะ (เช่น ทุกๆ 100ms) เพื่อประกาศตัวตน SSID, Channel และคุณสมบัติเครือข่าย |
| **Probe Request / Response** | Probe Request / Response   | Data / Frame      | แพ็กเกจคำร้องขอค้นหาจาก STA (Probe Request) และแพ็กเกจตอบรับข้อมูลเครือข่ายจาก AP (Probe Response)                 |
| **Authentication**           | IEEE 802.11 Authentication | Protocol / Phase  | กระบวนการยืนยันตัวตนระดับ Link Layer ก่อนขอผูกสัมพันธ์ (ใน WPA2 มักเป็นแบบ Open System)                            |
| **Association**              | IEEE 802.11 Association    | Protocol / Phase  | กระบวนการสถาปนาความสัมพันธ์และการตกลงพารามิเตอร์เครือข่าย (เช่น Supported Data Rates) ระหว่าง STA และ AP           |

---

## 6.3 คำศัพท์เกี่ยวกับระบบรักษาความปลอดภัยและการเข้ารหัส (Security & Encryption Keys)

| คำศัพท์ / ตัวย่อ       | คำเต็ม                 | มิติประเภท         | คำอธิบายและความหมาย                                                                                                   |
| :--------------------- | :--------------------- | :----------------- | :-------------------------------------------------------------------------------------------------------------------- |
| **WPA2 / WPA3**        | Wi-Fi Protected Access | Protocol Standard  | มาตรฐานความปลอดภัยเครือข่ายไร้สายที่ใช้การเข้ารหัสข้อมูลที่รัดกุม (AES / SAE)                                         |
| **PSK**                | Pre-Shared Key         | Data / Key         | รหัสผ่าน Wi-Fi (Passphrase) ที่ผู้ใช้กำหนดขึ้นและตกลงกันไว้ล่วงหน้า                                                   |
| **PMK**                | Pairwise Master Key    | Data / Key         | คีย์หลักระดับบนที่คำนวณถอดจาก PSK และ SSID ผ่านอัลกอริทึม PBKDF2                                                      |
| **PTK**                | Pairwise Transient Key | Data / Key         | คีย์เข้ารหัสข้อมูลแบบ Unicast (ระหว่าง ESP32 กับ AP เพียงคู่เดียว) สร้างจากการผสม PMK, ANonce, SNonce และ MAC Address |
| **GTK**                | Group Temporal Key     | Data / Key         | คีย์สำหรับเข้ารหัสข้อมูล Broadcast/Multicast สำหรับให้อุปกรณ์ทุกเครื่องใน AP อ่านข้อมูลกลุ่มได้                       |
| **ANonce**             | AP Nonce               | Data / Parameter   | ค่าตัวเลขสุ่มที่สร้างขึ้นใช้เพียงครั้งเดียว (Nonce) โดย AP ในขั้นตอน 1/4 EAPOL                                        |
| **SNonce**             | STA Nonce              | Data / Parameter   | ค่าตัวเลขสุ่มที่สร้างขึ้นใช้เพียงครั้งเดียว (Nonce) โดย Station (ESP32) ในขั้นตอน 2/4 EAPOL                           |
| **MIC**                | Message Integrity Code | Data / Checksum    | รหัสตรวจสอบความถูกต้องและความสมบูรณ์ของแพ็กเกจ เพื่อป้องกันการแก้ไขหรือปลอมแปลงรหัสผ่าน                               |
| **EAPOL**              | EAP over LAN           | Protocol Standard  | โปรโตคอลระดับ Layer 2 สำหรับส่งผ่านข้อความ Handshake ในการแลกเปลี่ยนคีย์รักษาความปลอดภัย                              |
| **Four-way Handshake** | Four-way Handshake     | Protocol Mechanism | กระบวนการแลกเปลี่ยนแพ็กเกจ 4 ขั้นตอนเพื่อยืนยันรหัสผ่านและติดตั้ง PTK/GTK โดยไม่ต้องส่งรหัสผ่านจริงผ่านคลื่นวิทยุ     |

---

## 6.4 คำศัพท์เกี่ยวกับสถาปัตยกรรมซอฟต์แวร์บน ESP32 (ESP32 & Networking Architecture)

| คำศัพท์ / ตัวย่อ | คำเต็ม                              | มิติประเภท         | คำอธิบายและความหมาย                                                                                                            |
| :--------------- | :---------------------------------- | :----------------- | :----------------------------------------------------------------------------------------------------------------------------- |
| **ESP-IDF**      | Espressif IoT Development Framework | Software Framework | คลังโปรแกรมและสภาพแวดล้อมการพัฒนาแอปพลิเคชันอย่างเป็นทางการของบริษัท Espressif สำหรับ ESP32                                    |
| **`esp_event`**  | ESP Event Loop Library              | Software Component | ระบบจัดการเหตุการณ์ (Event-driven Architecture) แบบ Asynchronous ภายใน ESP32 สำหรับรับรู้สถานะของ Wi-Fi/Network                |
| **`WIFI_EVENT`** | Wi-Fi Event Base                    | Data / Event ID    | กลุ่มเหตุการณ์ที่เกิดขึ้นในระดับ Wi-Fi Driver และ H/W Link Layer (เช่น `STA_START`, `STA_CONNECTED`, `STA_DISCONNECTED`)       |
| **`IP_EVENT`**   | IP Event Base                       | Data / Event ID    | กลุ่มเหตุการณ์ที่เกิดขึ้นในระดับ Network Layer / TCP/IP Stack (เช่น `STA_GOT_IP`, `STA_LOST_IP`)                               |
| **DHCP**         | Dynamic Host Configuration Protocol | Network Protocol   | โปรโตคอลที่แจกหมายเลข IP Address, Subnet Mask, Gateway และ DNS Server จาก AP ให้แก่ ESP32 โดยอัตโนมัติ                         |
| **lwIP**         | lightweight IP                      | Software Stack     | TCP/IP Stack ขนาดเล็กที่เป็น Open-source ถูกนำมาใช้ใน ESP32 เพื่อจัดการ Protocol เช่น IP, TCP, UDP, ICMP, DHCP                 |
| **Reason Code**  | Wi-Fi Disconnect Reason Code        | Data / Status Code | รหัสข้อผิดพลาดเชิงตัวเลข (เช่น 201 = `NO_AP_FOUND`, 15 = `HANDSHAKE_TIMEOUT`) ที่ Wi-Fi Driver แจ้งออกเมื่อการเชื่อมต่อล้มเหลว |
