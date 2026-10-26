# ใบงานที่ 5.2: การยืนยันตัวตน การสถาปนาการเชื่อมต่อ และการรับหมายเลข IP Address (Wi-Fi Connection & IP Assignment)

## 0. กล่าวนำ (Introduction)
ใบงานนี้เป็นการทดลองต่อเนื่องจากใบงานที่ 5.1 (Scan Phase) เพื่อศึกษาและสังเกตกระบวนการสถาปนาการเชื่อมต่อแบบครบวงจรในเฟสที่ 2 (Authentication), เฟสที่ 3 (Association), เฟสที่ 4 (4-Way Handshake) และเฟสที่ 5 (IP Assignment / DHCP) ผ่านเฟรมเวิร์ก ESP-IDF 

นักศึกษาจะได้วิเคราะห์พฤติกรรมของระบบและอ่านค่า Log สไตล์ Forensic เมื่อเกิดเหตุการณ์เชื่อมต่อสำเร็จ (`WIFI_EVENT_STA_CONNECTED`, `IP_EVENT_STA_GOT_IP`) รวมถึงการตรวจสอบและถอดรหัส Disconnect Reason Code (`WIFI_EVENT_STA_DISCONNECTED`) เมื่อเกิดเหตุการณ์เชื่อมต่อล้มเหลว (เช่น SSID ผิด หรือ Password ผิด)

---

## 1. วัตถุประสงค์ (Objectives)
1. เรียนรู้กระบวนการเชื่อมต่อ Wi-Fi และการจัดสรรหมายเลข IP Address (DHCP Client) ในโหมด Station (`WIFI_STA`) บน ESP-IDF
2. เรียนรู้การใช้ Event Loop (`esp_event_handler_instance_register`) และ FreeRTOS Event Group ในการดักจับ Event ของระบบ Wi-Fi และ IP
3. อ่านและวิเคราะห์โครงสร้างข้อมูล Event ได้แก่ `wifi_event_sta_connected_t`, `wifi_event_sta_disconnected_t` และ `ip_event_got_ip_t`
4. ตรวจสอบและระบุสาเหตุของความล้มเหลวในการเชื่อมต่อ Wi-Fi จากค่า Disconnect Reason Code (เช่น `WIFI_REASON_NO_AP_FOUND` และ `WIFI_REASON_HANDSHAKE_TIMEOUT` / `AUTH_FAIL`)

---

## 2. อุปกรณ์และซอฟต์แวร์ที่ใช้ในการทดลอง (Equipment & Tools)
1. บอร์ดไมโครคอนโทรลเลอร์ ESP32 (เช่น ESP32 DevKit V1) จำนวน 1 บอร์ด
2. สายเชื่อมต่อ Micro-USB หรือ USB-C จำนวน 1 เส้น
3. คอมพิวเตอร์ที่ติดตั้งโปรแกรม IDE เช่น VS Code พร้อมทั้ง ESP-IDF (อาจจะติดตั้งบนเครื่องหรือบน Docker ก็ได้)

---

## 3. ความรู้พื้นฐานที่เกี่ยวข้อง (Theoretical Background - ESP-IDF Framework)

### 3.1 สถาปัตยกรรม Event Loop และ Event Handling ใน ESP-IDF
ใน ESP-IDF การทำงานของ Wi-Fi เป็นแบบ Asynchronous (ทำงานเบื้องหลัง) โดย Driver จะส่ง Event ผ่านระบบ **Default Event Loop** เพื่อแจ้งเตือนให้โปรแกรมทราบความคืบหน้า

```mermaid
sequenceDiagram
    autonumber
    participant App as Application Code
    participant Evt as ESP Event Loop
    participant Drv as Wi-Fi Driver / IP Stack

    App->>Evt: esp_event_handler_instance_register()
    App->>Drv: esp_wifi_connect()
    Drv->>Evt: Post WIFI_EVENT_STA_CONNECTED
    Evt->>App: Callback: wifi_event_handler()
    Drv->>Evt: Post IP_EVENT_STA_GOT_IP
    Evt->>App: Callback: wifi_event_handler()
```

### 3.2 โครงสร้างข้อมูล Event สำคัญ (Class Diagrams)

#### 1) โครงสร้างข้อมูล `wifi_event_sta_connected_t`
ส่งมาพร้อมกับ Event `WIFI_EVENT_STA_CONNECTED` เพื่อระบุรายละเอียดของ AP ที่เชื่อมต่อสำเร็จ:

```mermaid
classDiagram
    class wifi_event_sta_connected_t {
        +uint8_t[33] ssid
        +uint8_t ssid_len
        +uint8_t[6] bssid
        +uint8_t channel
        +wifi_auth_mode_t authmode
        +uint16_t aid
    }
```

#### 2) โครงสร้างข้อมูล `wifi_event_sta_disconnected_t`
ส่งมาพร้อมกับ Event `WIFI_EVENT_STA_DISCONNECTED` เพื่อระบุสาเหตุของการหลุดการเชื่อมต่อ:

```mermaid
classDiagram
    class wifi_event_sta_disconnected_t {
        +uint8_t[33] ssid
        +uint8_t ssid_len
        +uint8_t[6] bssid
        +uint8_t reason
        +int8_t rssi
    }
```

#### 3) โครงสร้างข้อมูล `ip_event_got_ip_t`
ส่งมาพร้อมกับ Event `IP_EVENT_STA_GOT_IP` เมื่อ ESP32 ได้รับหมายเลข IP จาก DHCP Server:

```mermaid
classDiagram
    class ip_event_got_ip_t {
        +esp_ip4_addr_t ip
        +esp_ip4_addr_t netmask
        +esp_ip4_addr_t gw
        +bool ip_changed
    }
```

---

## 4. ขั้นตอนและโปรแกรมทดสอบการทดลอง (Experimental Procedures)

ในใบงานนี้ จะทำการทดสอบการเชื่อมต่อ Wi-Fi ใน 3 สถานการณ์ย่อย เพื่อเปรียบเทียบ Forensic Log และ Disconnect Reason Code:

### 5.2.1 การเชื่อมต่อด้วย SSID และ Password ที่ถูกต้อง (Success Case)
กำหนดค่า SSID และ Password ที่ถูกต้องตามสภาพแวดล้อมจริง สังเกต Event `WIFI_EVENT_STA_CONNECTED` และ `IP_EVENT_STA_GOT_IP` พร้อมอ่านหมายเลข IP Address, Subnet Mask และ Gateway

### 5.2.2 การเชื่อมต่อด้วย SSID ที่ไม่มีอยู่จริง (Wrong SSID / No AP Found)
กำหนดค่า SSID สมมุติที่ไม่มีอยู่จริง (`"NON_EXISTENT_SSID_9999"`) สังเกต Event `WIFI_EVENT_STA_DISCONNECTED` และวิเคราะห์ค่า Reason Code ซึ่งต้องได้ `WIFI_REASON_NO_AP_FOUND` (Decimal 201 / Hex `0xC9`)

### 5.2.3 การเชื่อมต่อด้วย SSID ที่ถูกต้องแต่ Password ผิด (Wrong Password / Handshake Fail)
กำหนดค่า SSID ถูกต้องแต่ป้อน Password ผิด (`"WRONG_PASS_9999"`) สังเกต Event `WIFI_EVENT_STA_DISCONNECTED` ในขั้นตอน 4-Way Handshake และวิเคราะห์ค่า Reason Code ซึ่งต้องได้ `WIFI_REASON_HANDSHAKE_TIMEOUT` (15) หรือ `WIFI_REASON_AUTH_FAIL` (202 / 204)

---

## 5. ซอร์สโค้ดการทดลอง (Complete ESP-IDF Source Code - `main.c`)

ให้นักศึกษานำซอร์สโค้ด C ต่อไปนี้ไปวางในไฟล์ `main/main.c` ของโปรเจกต์ ESP-IDF ทำการ Build และ Flash ลงบอร์ด ESP32 จากนั้นเปิด ESP-IDF Monitor (Baud Rate `115200`) เพื่อสังเกตผลการทำงาน

==**หมายเหตุ** ใน source code ด้านล่าง  แนะนำให้ใช้ MY_SSID และ  MY_PASSWORD จาก mobile hotspot และต้องลบออกก่อน push ขึ้น git== 

```c
#include <stdio.h>
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/event_groups.h"
#include "esp_system.h"
#include "esp_wifi.h"
#include "esp_event.h"
#include "esp_log.h"
#include "nvs_flash.h"
#include "esp_netif.h"

static const char *TAG = "LAB_WIFI_CONN";

/* FreeRTOS event group to signal when we are connected or failed */
static EventGroupHandle_t s_wifi_event_group;

#define WIFI_CONNECTED_BIT BIT0
#define WIFI_FAIL_BIT      BIT1

// Configurable target Wi-Fi credentials for successful test
#define EXAMPLE_ESP_WIFI_SSID      "MY_SSID"
#define EXAMPLE_ESP_WIFI_PASS      "MY_PASSWORD"

// Convert wifi_reason_code_t to readable string
static const char *get_disconnect_reason_name(uint8_t reason) {
  switch (reason) {
  case WIFI_REASON_UNSPECIFIED:
    return "WIFI_REASON_UNSPECIFIED (1)";
  case WIFI_REASON_AUTH_EXPIRE:
    return "WIFI_REASON_AUTH_EXPIRE (2)";
  case WIFI_REASON_AUTH_LEAVE:
    return "WIFI_REASON_AUTH_LEAVE (3)";
  case WIFI_REASON_ASSOC_EXPIRE:
    return "WIFI_REASON_ASSOC_EXPIRE (4)";
  case WIFI_REASON_ASSOC_FAIL:
    return "WIFI_REASON_ASSOC_FAIL (203)";
  case WIFI_REASON_NOT_AUTHED:
    return "WIFI_REASON_NOT_AUTHED (6)";
  case WIFI_REASON_HANDSHAKE_TIMEOUT:
    return "WIFI_REASON_HANDSHAKE_TIMEOUT (15)";
  case WIFI_REASON_NO_AP_FOUND:
    return "WIFI_REASON_NO_AP_FOUND (201)";
  case WIFI_REASON_AUTH_FAIL:
    return "WIFI_REASON_AUTH_FAIL (202)";
  case WIFI_REASON_4WAY_HANDSHAKE_TIMEOUT:
    return "WIFI_REASON_4WAY_HANDSHAKE_TIMEOUT (204)";
  case WIFI_REASON_CONNECTION_FAIL:
    return "WIFI_REASON_CONNECTION_FAIL (208)";
  case WIFI_REASON_BEACON_TIMEOUT:
    return "WIFI_REASON_BEACON_TIMEOUT (200)";
  default:
    return "OTHER_DISCONNECT_REASON";
  }
}

// Wi-Fi and IP Event Handler with Forensic Logging
static void wifi_event_handler(void *arg, esp_event_base_t event_base,
                               int32_t event_id, void *event_data) {
  if (event_base == WIFI_EVENT) {
    switch (event_id) {
    case WIFI_EVENT_STA_START:
      ESP_LOGI(TAG, "[EVENT FORENSIC]: WIFI_EVENT_STA_START received");
      ESP_LOGI(TAG, "[FORENSIC]: Call esp_wifi_connect()");
      esp_err_t err_conn = esp_wifi_connect();
      ESP_LOGI(TAG, "[FORENSIC]: esp_wifi_connect() returned %s (0x%x)",
               esp_err_to_name(err_conn), err_conn);
      break;

    case WIFI_EVENT_STA_CONNECTED: {
      wifi_event_sta_connected_t *event =
          (wifi_event_sta_connected_t *)event_data;
      ESP_LOGI(TAG, "=======================================================");
      ESP_LOGI(TAG, "[EVENT FORENSIC]: WIFI_EVENT_STA_CONNECTED received!");
      ESP_LOGI(TAG, "  -> Connected to SSID : %s", event->ssid);
      ESP_LOGI(TAG, "  -> BSSID            : %02X:%02X:%02X:%02X:%02X:%02X",
               event->bssid[0], event->bssid[1], event->bssid[2],
               event->bssid[3], event->bssid[4], event->bssid[5]);
      ESP_LOGI(TAG, "  -> Channel          : %d", event->channel);
      ESP_LOGI(TAG, "  -> Auth Mode        : %d", event->authmode);
      ESP_LOGI(TAG, "=======================================================");
      break;
    }

    case WIFI_EVENT_STA_DISCONNECTED: {
      wifi_event_sta_disconnected_t *event =
          (wifi_event_sta_disconnected_t *)event_data;
      ESP_LOGW(TAG, "=======================================================");
      ESP_LOGW(TAG, "[EVENT FORENSIC]: WIFI_EVENT_STA_DISCONNECTED received!");
      ESP_LOGW(TAG, "  -> Target SSID          : %s", event->ssid);
      ESP_LOGW(TAG, "  -> Reason Code (Decimal): %d", event->reason);
      ESP_LOGW(TAG, "  -> Reason Code (Hex)    : 0x%02X", event->reason);
      ESP_LOGW(TAG, "  -> Reason Description   : %s",
               get_disconnect_reason_name(event->reason));
      ESP_LOGW(TAG, "=======================================================");
      xEventGroupSetBits(s_wifi_event_group, WIFI_FAIL_BIT);
      break;
    }

    default:
      ESP_LOGI(TAG, "[EVENT FORENSIC]: WIFI_EVENT ID %ld received", event_id);
      break;
    }
  } else if (event_base == IP_EVENT) {
    if (event_id == IP_EVENT_STA_GOT_IP) {
      ip_event_got_ip_t *event = (ip_event_got_ip_t *)event_data;
      ESP_LOGI(TAG, "=======================================================");
      ESP_LOGI(TAG, "[EVENT FORENSIC]: IP_EVENT_STA_GOT_IP received!");
      ESP_LOGI(TAG, "  -> IP Address : " IPSTR, IP2STR(&event->ip_info.ip));
      ESP_LOGI(TAG, "  -> Netmask    : " IPSTR, IP2STR(&event->ip_info.netmask));
      ESP_LOGI(TAG, "  -> Gateway    : " IPSTR, IP2STR(&event->ip_info.gw));
      ESP_LOGI(TAG, "=======================================================");
      xEventGroupSetBits(s_wifi_event_group, WIFI_CONNECTED_BIT);
    }
  }
}

// Function to test Wi-Fi connection with specific config
static void test_wifi_connection(const char *test_title, const char *ssid,
                                  const char *password) {
  ESP_LOGI(TAG, "\n");
  ESP_LOGI(TAG, "------------------------------------------------------------------");
  ESP_LOGI(TAG, ">>> %s", test_title);
  ESP_LOGI(TAG, "------------------------------------------------------------------");
  ESP_LOGI(TAG, "  Target SSID: \"%s\"", ssid);
  ESP_LOGI(TAG, "  Target Password: \"%s\"", password);

  // Clear event bits
  xEventGroupClearBits(s_wifi_event_group, WIFI_CONNECTED_BIT | WIFI_FAIL_BIT);

  wifi_config_t wifi_config = {
      .sta = {
          .threshold.authmode = WIFI_AUTH_WPA2_PSK,
      },
  };
  strncpy((char *)wifi_config.sta.ssid, ssid, sizeof(wifi_config.sta.ssid));
  strncpy((char *)wifi_config.sta.password, password,
          sizeof(wifi_config.sta.password));

  ESP_LOGI(TAG, "[FORENSIC]: Call esp_wifi_stop()");
  esp_wifi_stop();

  ESP_LOGI(TAG, "[FORENSIC]: Call esp_wifi_set_config(WIFI_IF_STA, &wifi_config)");
  esp_err_t err_cfg = esp_wifi_set_config(WIFI_IF_STA, &wifi_config);
  ESP_LOGI(TAG, "[FORENSIC]: esp_wifi_set_config() returned %s (0x%x)",
           esp_err_to_name(err_cfg), err_cfg);

  ESP_LOGI(TAG, "[FORENSIC]: Call esp_wifi_start()");
  esp_err_t err_start = esp_wifi_start();
  ESP_LOGI(TAG, "[FORENSIC]: esp_wifi_start() returned %s (0x%x)",
           esp_err_to_name(err_start), err_start);

  /* Waiting until either the connection is established (WIFI_CONNECTED_BIT) or failed (WIFI_FAIL_BIT) */
  EventBits_t bits = xEventGroupWaitBits(s_wifi_event_group,
                                         WIFI_CONNECTED_BIT | WIFI_FAIL_BIT,
                                         pdFALSE, pdFALSE, pdMS_TO_TICKS(10000));

  if (bits & WIFI_CONNECTED_BIT) {
    ESP_LOGI(TAG, "[RESULT]: TEST PASSED - Connected to AP successfully!");
  } else if (bits & WIFI_FAIL_BIT) {
    ESP_LOGW(TAG, "[RESULT]: TEST FAILED - Disconnected event captured.");
  } else {
    ESP_LOGE(TAG, "[RESULT]: TEST TIMEOUT - Neither connected nor disconnected event received.");
  }
}

void app_main(void) {
  s_wifi_event_group = xEventGroupCreate();

  // 1. Initialize NVS Flash
  ESP_LOGI(TAG, "[FORENSIC]: Call nvs_flash_init()");
  esp_err_t ret = nvs_flash_init();
  ESP_LOGI(TAG, "[FORENSIC]: nvs_flash_init() returned %s (0x%x)",
           esp_err_to_name(ret), ret);
  if (ret == ESP_ERR_NVS_NO_FREE_PAGES ||
      ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
    ESP_LOGI(TAG, "[FORENSIC]: Call nvs_flash_erase()");
    ESP_ERROR_CHECK(nvs_flash_erase());
    ret = nvs_flash_init();
    ESP_LOGI(TAG, "[FORENSIC]: nvs_flash_init() retry returned %s (0x%x)",
             esp_err_to_name(ret), ret);
  }
  ESP_ERROR_CHECK(ret);

  // 2. Initialize Network Interface and Event Loop
  ESP_LOGI(TAG, "[FORENSIC]: Call esp_netif_init()");
  ESP_ERROR_CHECK(esp_netif_init());

  ESP_LOGI(TAG, "[FORENSIC]: Call esp_event_loop_create_default()");
  ESP_ERROR_CHECK(esp_event_loop_create_default());

  ESP_LOGI(TAG, "[FORENSIC]: Call esp_netif_create_default_wifi_sta()");
  esp_netif_t *sta_netif = esp_netif_create_default_wifi_sta();
  ESP_LOGI(TAG, "[FORENSIC]: esp_netif_create_default_wifi_sta() returned %p", sta_netif);

  // 3. Initialize Wi-Fi Driver
  wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
  ESP_LOGI(TAG, "[FORENSIC]: Call esp_wifi_init(&cfg)");
  ESP_ERROR_CHECK(esp_wifi_init(&cfg));

  // 4. Register Event Handlers
  esp_event_handler_instance_t instance_any_id;
  esp_event_handler_instance_t instance_got_ip;
  ESP_LOGI(TAG, "[FORENSIC]: Call esp_event_handler_instance_register(WIFI_EVENT)");
  ESP_ERROR_CHECK(esp_event_handler_instance_register(
      WIFI_EVENT, ESP_EVENT_ANY_ID, &wifi_event_handler, NULL,
      &instance_any_id));

  ESP_LOGI(TAG, "[FORENSIC]: Call esp_event_handler_instance_register(IP_EVENT)");
  ESP_ERROR_CHECK(esp_event_handler_instance_register(
      IP_EVENT, IP_EVENT_STA_GOT_IP, &wifi_event_handler, NULL,
      &instance_got_ip));

  ESP_LOGI(TAG, "[FORENSIC]: Call esp_wifi_set_mode(WIFI_MODE_STA)");
  ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));

  ESP_LOGI(TAG, "==================================================================");
  ESP_LOGI(TAG, "  Lab 5.2: Wi-Fi Connection & IP Assignment (ESP-IDF Forensic)");
  ESP_LOGI(TAG, "==================================================================");

  // ------------------------------------------------------------------
  // 5.2.1 Connecting with Correct SSID & Password (Success Case)
  // ------------------------------------------------------------------
  test_wifi_connection("Experiment 5.2.1: Connection Test - Correct Credentials",
                       EXAMPLE_ESP_WIFI_SSID, EXAMPLE_ESP_WIFI_PASS);

  vTaskDelay(pdMS_TO_TICKS(2000));

  // ------------------------------------------------------------------
  // 5.2.2 Connecting with Wrong SSID (Non-existent AP Case)
  // ------------------------------------------------------------------
  test_wifi_connection("Experiment 5.2.2: Connection Test - Wrong SSID (No AP Found)",
                       "NON_EXISTENT_SSID_9999", "12345678");

  vTaskDelay(pdMS_TO_TICKS(2000));

  // ------------------------------------------------------------------
  // 5.2.3 Connecting with Correct SSID but Wrong Password (Handshake Fail Case)
  // ------------------------------------------------------------------
  test_wifi_connection("Experiment 5.2.3: Connection Test - Wrong Password (Auth/Handshake Fail)",
                       EXAMPLE_ESP_WIFI_SSID, "WRONG_PASS_9999");

  ESP_LOGI(TAG, "==================================================================");
  ESP_LOGI(TAG, "  [Phase 2/3/4/5 Completed: Wi-Fi Connection Lab Finished]");
  ESP_LOGI(TAG, "==================================================================");
}
```

---

## 6. ตารางบันทึกผลการทดลอง (Experiment Results)

ให้นักศึกษาบันทึกผลลัพธ์จากการสังเกตใน Serial Console ลงในตารางต่อไปนี้:

### 6.1 ตารางสรุปเปรียบเทียบผลการทดลองทั้ง 3 สถานการณ์

| ข้อการทดลอง | สถานการณ์ทดสอบ | Event สุดท้ายที่ได้รับ | ผลลัพธ์ (Passed/Failed) | Reason Code (Decimal / Hex) | คำอธิบาย Reason Code |
| :---: | :--- | :---: | :---: | :---: | :--- |
| **5.2.1** | SSID และ Password ถูกต้อง |WIFI_EVENT_STA_CONNECTED|Connected |-|Connected to AP successfully!|
| **5.2.2** | ระบุ SSID ผิด (ไม่มีในระบบ) |WIFI_EVENT_STA_DISCONNECTED received!|Failed|8/0x08|OTHER_DISCONNECT_REASON|
| **5.2.3** | ระบุ SSID ถูกต้อง แต่ Password ผิด |WIFI_EVENT_STA_DISCONNECTED|Failed|15/0x0F|WIFI_REASON_4WAY_HANDSHAKE_TIMEOUT (204)|

### 6.2 บันทึกข้อมูลเครือข่ายจากการเชื่อมต่อสำเร็จ (ข้อ 5.2.1)

| พารามิเตอร์เครือข่าย | ค่าที่ได้รับจริงจาก DHCP |
| :--- | :--- |
| **SSID** |😑|
| **BSSID (MAC Address)** |62:DD:AB:3C:2D:A6|
| **Channel** |11|
| **IP Address** |10.156.141.4|
| **Subnet Mask** |255.255.255.0|
| **Default Gateway** |10.156.141.184|

---

## 7. คำถามท้ายการทดลอง (Post-Lab Questions)

1. เหตุใดการระบุ SSID ผิด (ข้อ 5.2.2) จึงส่งผลให้เกิด Disconnect Event ด้วย Reason Code `201` (`WIFI_REASON_NO_AP_FOUND`) ตั้งแต่เฟส Scan?
```
เนื่องจากกลไกของ Wi-Fi Station (STA) เมื่อได้รับคำสั่ง esp_wifi_connect() บอร์ดจะเริ่มทำการ Active Scanning (ส่ง Probe Request ออกไปทุก Channel ที่รองรับ) เพื่อค้นหา Access Point ที่มีชื่อ SSID ตรงกับโครงสร้างอินพุต เมื่อสแกนจนครบทุกย่านความถี่แล้วไม่พบสัญญาณ Beacon หรือ Probe Response จาก AP ที่มีชื่อ "NON_EXISTENT_SSID_9999" ระบบจึงไม่สามารถเข้าสู่เฟส Authentication ได้ และยุติการทำงานลงทันที พร้อมแจ้ง Wi-Fi Driver Subsystem ให้ส่ง Event Disconnected ด้วยรหัสความล้มเหลว 201 (ไม่พบเป้าหมาย)
```
2. เหตุใดการพิมพ์ Password ผิด (ข้อ 5.2.3) จึงผ่านเฟส Auth และ Assoc มาได้ แต่มาล้มเหลวในเฟส 4-Way Handshake (Reason Code `15` หรือ `204`)?
```
ในสถาปัตยกรรมความปลอดภัย WPA2-PSK กระบวนการในเฟส Authentication และ Association ช่วงแรกเป็นเพียงการจับคู่ระดับโครงสร้างพื้นฐานทางกายภาพ (Open System Authentication) ว่าคุยภาษาเดียวกันและยอมรับโหนดเข้าสู่ระบบเครือข่ายหรือไม่

แต่ขั้นตอนการตรวจสอบความถูกต้องของรหัสผ่านผ่านการพิสูจน์ตัวตน (Authentication) จริง ๆ จะเกิดขึ้นใน Layer ถัดมาคือ 4-Way Handshake เพื่อสร้างและยืนยันคีย์เข้ารหัสลับร่วมกัน (Pairwise Transient Key - PTK) โดยคำนวณจาก Pre-Shared Key (PSK) ที่แปลงมาจาก Password เมื่อเราป้อน Password ผิด ตัวบอร์ดและ AP จะคำนวณค่าคีย์ไม่ตรงกัน ทำให้การแลกเปลี่ยนข้อความ EAPOL (Message 2 หรือ 3) ไม่ถูกต้อง ส่งผลให้เกิดการรอคอยจนหมดเวลา และถูกตัดการเชื่อมต่อด้วยข้อผิดพลาดกลุ่ม Handshake Timeout ในที่สุด
```
3. ลำดับการเกิด Event ระหว่าง **`WIFI_EVENT_STA_CONNECTED`** กับ **`IP_EVENT_STA_GOT_IP`** Event ใดเกิดขึ้นก่อนกัน และมีความหมายทางกายภาพของ Layer
Network ต่างกันอย่างไร?
```
WIFI_EVENT_STA_CONNECTED เกิดขึ้นก่อน: มีความหมายใน Layer 2 (Data Link Layer) บอร์ด ESP32 ได้ทำการตกลงเข้ารหัสสัญญาณวิทยุ ผูกความสัมพันธ์ (Associate) และทำ 4-Way Handshake กับ Access Point เสร็จสิ้นทางฮาร์ดแวร์ไร้สายเรียบร้อยแล้ว

IP_EVENT_STA_GOT_IP เกิดขึ้นทีหลัง: มีความหมายใน Layer 3 (Network Layer) หลังจาก Layer 2 เชื่อมต่อสำเร็จ ซอฟต์แวร์ Network Stack (LWIP) จะเริ่มทำการส่งแพ็กเกจ DHCP Request เพื่อขอรับการจัดสรรหมายเลข Logical Address (IP Address, Subnet, Gateway) จาก Server เมื่อได้รับไอพีกลับมาใช้งานสื่อสารข้ามเครือข่ายอินเทอร์เน็ตได้แล้ว จึงเกิด Event นี้ขึ้น
```
4. สมาชิกตัวแปร `reason` ในโครงสร้าง `wifi_event_sta_disconnected_t` มีประโยชน์อย่างไรต่อการออกแบบระบบค้นหาสาเหตุและกู้คืนการเชื่อมต่อ (Auto-Reconnection Mechanism) ในแอปพลิเคชัน IoT?
```
ตัวแปร reason มีความสำคัญอย่างมากในการทำ Fault Tolerance และสร้าง State Machine เพื่อกู้คืนระบบอัตโนมัติ (Auto-Reconnection) อย่างมีประสิทธิภาพ เพราะทำให้นักพัฒนาสามารถแยกแยะปัญหาและเขียน Logic รับมือเฉพาะเจาะจงได้
- ถ้าหลุดด้วยสาเหตุสัญญาณอ่อนหรือ AP ดับชั่วคราว เช่น WIFI_REASON_BEACON_TIMEOUT หรือ WIFI_REASON_NO_AP_FOUND: ให้ระบบหน่วงเวลา (Exponential Backoff) แล้วสั่ง esp_wifi_connect() เพื่อพยายามเชื่อมต่อใหม่เรื่อย ๆ
- ถ้าหลุดด้วยสาเหตุระบบความปลอดภัย เช่น WIFI_REASON_HANDSHAKE_TIMEOUT หรือ WIFI_REASON_AUTH_FAIL: แสดงว่า Password ผิดพลาด ควรหยุดการเชื่อมต่อใหม่ซ้ำ ๆ เพื่อป้องกันการวนลูปไม่รู้จบ (ซึ่งจะเปลืองพลังงานและ CPU) แล้วเปลี่ยนโหมดบอร์ดให้เข้าสู่ SmartConfig หรือ AP Mode เพื่อให้ผู้ใช้ป้อน Credential ชุดใหม่แทน
```
** command log **

```
rst:0x1 (POWERON_RESET),boot:0x13 (S0x1 (POWERON_RESET),boot:0x13 (S0x1 (POWERON_RESET),boot:0x13 (S0x1 (POWERON_RESET),boot:0x13 (S0x1 (POWERON_RESET),boot:0x13 (S0x1 (POWERON_RESET),boot:0x13 (S0x1 (POWERON_RESET),boot:0x13 (S0x1 (POWERON_RESET),boot:0x13 (S0x1 (POWERON_RESET),boot:0x13 (S0x1 (POWERON_RESET),boot:0x13 (S0x1 (POWERON_RESET),boot:0x13 (S0x1 (POWERON_RESET),boot:0x13 (S0x1 (POWERON_RESET),boot:0x13 (S0x1 (POWERON_RESET),boot:0x13 (S0x1 (POWERON_RESET),boot:0x13 (S0x1 (POWERON_RESET),boot:0x13 (S0x1 (POWERON_RESET),boot:0x13 (S0x1 (POWERON_RESET),boot:0x13 (S0x1 (POWERON_RESET),boot:0x13 (S0x1 (POWERON_RESET),boot:0x13 (S0x1 (POWERON_RESET),boot:0x13 (S0x1 (POWERON_RESET),boot:0x13 (S0x1 (POWERON_RESET),boot:0x13 (S0x1 (POWERON_RESET),boot:0x13 (S0x1 (POWERON_RESET),boot:0x13 (S0x1 (POWERON_RESET),boot:0x13 (S0x1 (POWERON_RESET),boot:0x13 (S0x1 (POWERON_RESET),boot:0x13 (S0x1 (POWERON_RESET),boot:0x13 (S0x1 (POWERON_RESET),boot:0x13 (S0x1 (POWERON_RESET),boot:0x13 (SPI_FAST_FLASH_BOOT)
configsip: 0, SPIWP:0xee
clk_drv:0x00,q_drv:0x00,d_drv:0x00,cs0_drv:0x00,hd_drv:0x00,wp_drv:0x00
mode:DIO, clock div:2
load:0x3fff0030,len:6380
ho 0 tail 12 room 4
load:0x40078000,len:15916
load:0x40080400,len:3860
--- 0x40080400: _invalid_pc_placeholder at /Users/cheen/esp/v5.5.1/esp-idf/components/xtensa/xtensa_vectors.S:2235
entry 0x40080638
--- 0x40080638: call_start_cpu0 at /Users/cheen/esp/v5.5.1/esp-idf/components/bootloader/subproject/main/bootloader_start.c:25
I (29) boot: ESP-IDF v5.5.1-dirty 2nd stage bootloader
I (29) boot: compile time Oct 26 2026 10:42:53
I (29) boot: Multicore bootloader
I (31) boot: chip revision: v3.1
I (34) boot.esp32: SPI Speed      : 40MHz
I (38) boot.esp32: SPI Mode       : DIO
I (41) boot.esp32: SPI Flash Size : 2MB
I (45) boot: Enabling RNG early entropy source...
I (49) boot: Partition Table:
I (52) boot: ## Label            Usage          Type ST Offset   Length
I (58) boot:  0 nvs              WiFi data        01 02 00009000 00006000
I (65) boot:  1 phy_init         RF data          01 01 0000f000 00001000
I (71) boot:  2 factory          factory app      00 00 00010000 00100000
I (78) boot: End of partition table
I (81) esp_image: segment 0: paddr=00010020 vaddr=3f400020 size=19e38h (106040) map
I (125) esp_image: segment 1: paddr=00029e60 vaddr=3ffb0000 size=03eech ( 16108) load
I (131) esp_image: segment 2: paddr=0002dd54 vaddr=40080000 size=022c4h (  8900) load
I (135) esp_image: segment 3: paddr=00030020 vaddr=400d0020 size=861fch (549372) map
I (324) esp_image: segment 4: paddr=000b6224 vaddr=400822c4 size=15b48h ( 88904) load
I (359) esp_image: segment 5: paddr=000cbd74 vaddr=50000000 size=00020h (    32) load
I (371) boot: Loaded app from partition at offset 0x10000
I (371) boot: Disabling RNG early entropy source...
I (381) cpu_start: Multicore app
I (390) cpu_start: Pro cpu start user code
I (390) cpu_start: cpu freq: 160000000 Hz
I (390) app_init: Application information:
I (390) app_init: Project name:     wifi_connection_phase
I (395) app_init: App version:      15a2e44-dirty
I (399) app_init: Compile time:     Oct 26 2026 10:42:40
I (404) app_init: ELF file SHA256:  5fb069f56...
I (409) app_init: ESP-IDF:          v5.5.1-dirty
I (413) efuse_init: Min chip rev:     v0.0
I (417) efuse_init: Max chip rev:     v3.99 
I (421) efuse_init: Chip rev:         v3.1
I (425) heap_init: Initializing. RAM available for dynamic allocation:
I (431) heap_init: At 3FFAE6E0 len 00001920 (6 KiB): DRAM
I (436) heap_init: At 3FFB7FD0 len 00028030 (160 KiB): DRAM
I (441) heap_init: At 3FFE0440 len 00003AE0 (14 KiB): D/IRAM
I (447) heap_init: At 3FFE4350 len 0001BCB0 (111 KiB): D/IRAM
I (452) heap_init: At 40097E0C len 000081F4 (32 KiB): IRAM
I (459) spi_flash: detected chip: generic
I (461) spi_flash: flash io: dio
W (464) spi_flash: Detected size(4096k) larger than the size in the binary image header(2048k). Using the size in the binary image header.
I (477) main_task: Started on CPU0
I (487) main_task: Calling app_main()
I (487) LAB_WIFI_CONN: [FORENSIC]: Call nvs_flash_init()
I (517) LAB_WIFI_CONN: [FORENSIC]: nvs_flash_init() returned ESP_OK (0x0)
I (517) LAB_WIFI_CONN: [FORENSIC]: Call esp_netif_init()
I (517) LAB_WIFI_CONN: [FORENSIC]: Call esp_event_loop_create_default()
I (527) LAB_WIFI_CONN: [FORENSIC]: Call esp_netif_create_default_wifi_sta()
I (527) LAB_WIFI_CONN: [FORENSIC]: esp_netif_create_default_wifi_sta() returned 0x3ffbd940
I (537) LAB_WIFI_CONN: [FORENSIC]: Call esp_wifi_init(&cfg)
I (557) wifi:wifi driver task: 3ffc002c, prio:23, stack:6656, core=0
I (577) wifi:wifi firmware version: 14da9b7
I (577) wifi:wifi certification version: v7.0
I (577) wifi:config NVS flash: enabled
I (577) wifi:config nano formatting: disabled
I (577) wifi:Init data frame dynamic rx buffer num: 32
I (587) wifi:Init static rx mgmt buffer num: 5
I (587) wifi:Init management short buffer num: 32
I (587) wifi:Init dynamic tx buffer num: 32
I (597) wifi:Init static rx buffer size: 1600
I (597) wifi:Init static rx buffer num: 10
I (607) wifi:Init dynamic rx buffer num: 32
I (607) wifi_init: rx ba win: 6
I (607) wifi_init: accept mbox: 6
I (617) wifi_init: tcpip mbox: 32
I (617) wifi_init: udp mbox: 6
I (617) wifi_init: tcp mbox: 6
I (617) wifi_init: tcp tx win: 5760
I (627) wifi_init: tcp rx win: 5760
I (627) wifi_init: tcp mss: 1440
I (627) wifi_init: WiFi IRAM OP enabled
I (637) wifi_init: WiFi RX IRAM OP enabled
I (637) LAB_WIFI_CONN: [FORENSIC]: Call esp_event_handler_instance_register(WIFI_EVENT)
I (647) LAB_WIFI_CONN: [FORENSIC]: Call esp_event_handler_instance_register(IP_EVENT)
I (657) LAB_WIFI_CONN: [FORENSIC]: Call esp_wifi_set_mode(WIFI_MODE_STA)
I (657) LAB_WIFI_CONN: ==================================================================
I (667) LAB_WIFI_CONN:   Lab 5.2: Wi-Fi Connection & IP Assignment (ESP-IDF Forensic)
I (677) LAB_WIFI_CONN: ==================================================================
I (687) LAB_WIFI_CONN: 

I (687) LAB_WIFI_CONN: ------------------------------------------------------------------
I (697) LAB_WIFI_CONN: >>> Experiment 5.2.1: Connection Test - Correct Credentials
I (697) LAB_WIFI_CONN: ------------------------------------------------------------------
I (707) LAB_WIFI_CONN:   Target SSID: "😑"
I (707) LAB_WIFI_CONN:   Target Password: "yoec6766"
I (717) LAB_WIFI_CONN: [FORENSIC]: Call esp_wifi_stop()
I (717) LAB_WIFI_CONN: [FORENSIC]: Call esp_wifi_set_config(WIFI_IF_STA, &wifi_config)
I (767) LAB_WIFI_CONN: [FORENSIC]: esp_wifi_set_config() returned ESP_OK (0x0)
I (767) LAB_WIFI_CONN: [FORENSIC]: Call esp_wifi_start()
I (777) phy_init: phy_version 4861,b71b5ad,Aug  5 2025,11:16:06
I (847) wifi:mode : sta (84:1f:e8:39:c0:80)
I (847) wifi:enable tsf
I (857) LAB_WIFI_CONN: [EVENT FORENSIC]: WIFI_EVENT ID 43 received
I (857) LAB_WIFI_CONN: [FORENSIC]: esp_wifi_start() returned ESP_OK (0x0)
I (857) LAB_WIFI_CONN: [EVENT FORENSIC]: WIFI_EVENT_STA_START received
I (867) LAB_WIFI_CONN: [FORENSIC]: Call esp_wifi_connect()
I (877) LAB_WIFI_CONN: [FORENSIC]: esp_wifi_connect() returned ESP_OK (0x0)
I (1137) wifi:new:<11,0>, old:<1,0>, ap:<255,255>, sta:<11,0>, prof:1, snd_ch_cfg:0x0
I (1147) wifi:state: init -> auth (0xb0)
I (1147) LAB_WIFI_CONN: [EVENT FORENSIC]: WIFI_EVENT ID 43 received
I (1177) wifi:state: auth -> assoc (0x0)
I (1187) wifi:state: assoc -> run (0x10)
I (1237) wifi:connected with 😑, aid = 18, channel 11, BW20, bssid = 62:dd:ab:3c:2d:a6
I (1237) wifi:security: WPA2-PSK, phy: bgn, rssi: -47
I (1247) wifi:pm start, type: 1

I (1247) wifi:dp: 1, bi: 102400, li: 3, scale listen interval from 307200 us to 307200 us
I (1247) LAB_WIFI_CONN: =======================================================
I (1257) LAB_WIFI_CONN: [EVENT FORENSIC]: WIFI_EVENT_STA_CONNECTED received!
I (1257) LAB_WIFI_CONN:   -> Connected to SSID : 😑
I (1267) LAB_WIFI_CONN:   -> BSSID            : 62:DD:AB:3C:2D:A6
I (1277) LAB_WIFI_CONN:   -> Channel          : 11
I (1277) LAB_WIFI_CONN:   -> Auth Mode        : 3
I (1277) LAB_WIFI_CONN: =======================================================
I (1317) wifi:dp: 2, bi: 102400, li: 4, scale listen interval from 307200 us to 409600 us
I (1327) wifi:AP's beacon interval = 102400 us, DTIM period = 2
I (2267) esp_netif_handlers: sta ip: 10.156.141.4, mask: 255.255.255.0, gw: 10.156.141.184
I (2267) LAB_WIFI_CONN: =======================================================
I (2267) LAB_WIFI_CONN: [EVENT FORENSIC]: IP_EVENT_STA_GOT_IP received!
I (2277) LAB_WIFI_CONN:   -> IP Address : 10.156.141.4
I (2277) LAB_WIFI_CONN:   -> Netmask    : 255.255.255.0
I (2287) LAB_WIFI_CONN:   -> Gateway    : 10.156.141.184
I (2287) LAB_WIFI_CONN: =======================================================
I (2297) LAB_WIFI_CONN: [RESULT]: TEST PASSED - Connected to AP successfully!
I (4307) LAB_WIFI_CONN: 

I (4307) LAB_WIFI_CONN: ------------------------------------------------------------------
I (4307) LAB_WIFI_CONN: >>> Experiment 5.2.2: Connection Test - Wrong SSID (No AP Found)
I (4307) LAB_WIFI_CONN: ------------------------------------------------------------------
I (4317) LAB_WIFI_CONN:   Target SSID: "NON_EXISTENT_SSID_9999"
I (4327) LAB_WIFI_CONN:   Target Password: "12345678"
I (4327) LAB_WIFI_CONN: [FORENSIC]: Call esp_wifi_stop()
I (4337) wifi:state: run -> init (0x0)
I (4347) wifi:pm stop, total sleep time: 2544147 us / 3098018 us

I (4347) wifi:new:<11,0>, old:<11,0>, ap:<255,255>, sta:<11,0>, prof:1, snd_ch_cfg:0x0
W (4357) LAB_WIFI_CONN: =======================================================
W (4357) LAB_WIFI_CONN: [EVENT FORENSIC]: WIFI_EVENT_STA_DISCONNECTED received!
W (4367) LAB_WIFI_CONN:   -> Target SSID          : 😑
W (4367) LAB_WIFI_CONN:   -> Reason Code (Decimal): 8
W (4377) LAB_WIFI_CONN:   -> Reason Code (Hex)    : 0x08
W (4377) LAB_WIFI_CONN:   -> Reason Description   : OTHER_DISCONNECT_REASON
W (4387) LAB_WIFI_CONN: =======================================================
I (4397) LAB_WIFI_CONN: [EVENT FORENSIC]: WIFI_EVENT ID 3 received
I (4397) wifi:flush txq
I (4397) wifi:stop sw txq
I (4407) wifi:lmac stop hw txq
I (4407) LAB_WIFI_CONN: [FORENSIC]: Call esp_wifi_set_config(WIFI_IF_STA, &wifi_config)
I (4457) LAB_WIFI_CONN: [FORENSIC]: esp_wifi_set_config() returned ESP_OK (0x0)
I (4457) LAB_WIFI_CONN: [FORENSIC]: Call esp_wifi_start()
I (4467) wifi:mode : sta (84:1f:e8:39:c0:80)
I (4467) wifi:enable tsf
I (4467) LAB_WIFI_CONN: [EVENT FORENSIC]: WIFI_EVENT ID 43 received
I (4477) LAB_WIFI_CONN: [FORENSIC]: esp_wifi_start() returned ESP_OK (0x0)
I (4477) LAB_WIFI_CONN: [EVENT FORENSIC]: WIFI_EVENT_STA_START received
I (4487) LAB_WIFI_CONN: [FORENSIC]: Call esp_wifi_connect()
I (4497) LAB_WIFI_CONN: [FORENSIC]: esp_wifi_connect() returned ESP_OK (0x0)
W (4497) LAB_WIFI_CONN: [RESULT]: TEST FAILED - Disconnected event captured.
I (6507) LAB_WIFI_CONN: 

I (6507) LAB_WIFI_CONN: ------------------------------------------------------------------
I (6507) LAB_WIFI_CONN: >>> Experiment 5.2.3: Connection Test - Wrong Password (Auth/Handshake Fail)
I (6507) LAB_WIFI_CONN: ------------------------------------------------------------------
I (6517) LAB_WIFI_CONN:   Target SSID: "😑"
I (6527) LAB_WIFI_CONN:   Target Password: "WRONG_PASS_9999"
I (6527) LAB_WIFI_CONN: [FORENSIC]: Call esp_wifi_stop()
I (6537) LAB_WIFI_CONN: [EVENT FORENSIC]: WIFI_EVENT ID 3 received
I (6537) wifi:flush txq
I (6537) wifi:stop sw txq
I (6547) wifi:lmac stop hw txq
I (6547) LAB_WIFI_CONN: [FORENSIC]: Call esp_wifi_set_config(WIFI_IF_STA, &wifi_config)
I (6607) LAB_WIFI_CONN: [FORENSIC]: esp_wifi_set_config() returned ESP_OK (0x0)
I (6607) LAB_WIFI_CONN: [FORENSIC]: Call esp_wifi_start()
I (6607) wifi:mode : sta (84:1f:e8:39:c0:80)
I (6607) wifi:enable tsf
I (6617) LAB_WIFI_CONN: [EVENT FORENSIC]: WIFI_EVENT_STA_START received
I (6617) LAB_WIFI_CONN: [FORENSIC]: Call esp_wifi_connect()
I (6627) LAB_WIFI_CONN: [FORENSIC]: esp_wifi_connect() returned ESP_OK (0x0)
I (6617) LAB_WIFI_CONN: [FORENSIC]: esp_wifi_start() returned ESP_OK (0x0)
I (6907) wifi:new:<11,0>, old:<1,0>, ap:<255,255>, sta:<11,0>, prof:1, snd_ch_cfg:0x0
I (6907) wifi:state: init -> auth (0xb0)
I (6917) LAB_WIFI_CONN: [EVENT FORENSIC]: WIFI_EVENT ID 43 received
I (6997) wifi:state: auth -> assoc (0x0)
I (7017) wifi:state: assoc -> run (0x10)
I (10087) wifi:state: run -> init (0xf00)
I (10097) wifi:new:<11,0>, old:<11,0>, ap:<255,255>, sta:<11,0>, prof:1, snd_ch_cfg:0x0
W (10097) LAB_WIFI_CONN: =======================================================
W (10097) LAB_WIFI_CONN: [EVENT FORENSIC]: WIFI_EVENT_STA_DISCONNECTED received!
W (10107) LAB_WIFI_CONN:   -> Target SSID          : 😑
W (10107) LAB_WIFI_CONN:   -> Reason Code (Decimal): 15
W (10117) LAB_WIFI_CONN:   -> Reason Code (Hex)    : 0x0F
W (10117) LAB_WIFI_CONN:   -> Reason Description   : WIFI_REASON_4WAY_HANDSHAKE_TIMEOUT (204)
W (10127) LAB_WIFI_CONN: =======================================================
W (10137) LAB_WIFI_CONN: [RESULT]: TEST FAILED - Disconnected event captured.
I (10147) LAB_WIFI_CONN: ==================================================================
I (10157) LAB_WIFI_CONN:   [Phase 2/3/4/5 Completed: Wi-Fi Connection Lab Finished]
I (10157) LAB_WIFI_CONN: ==================================================================
I (10167) main_task: Returned from app_main()
```
