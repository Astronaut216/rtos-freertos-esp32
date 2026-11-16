# Project cá nhân (70%) – FreeRTOS trên ESP32

Sinh viên: **Đặng Đình Mạnh** – MSSV **24119055** · GVHD: Huỳnh Hoàng Hà
Báo cáo: [`../03_Reports/Report_Project_CaNhan_FreeRTOS_24119055.pdf`](../03_Reports/Report_Project_CaNhan_FreeRTOS_24119055.pdf)

![Kiến trúc phần mềm](docs/figures/f2_software.png)

## Phần cứng (kit chạy thật: ESP32 DevKit WROOM-32, env esp32dev, COM5)

| Linh kiện | Chân ESP32 |
|---|---|
| LED1 / LED2 / LED3 (+220 Ω) | GPIO19 / 18 / 23 |
| BTN1 / BTN2 (nối GND, pull-up trong) | GPIO4 / 13 |
| Quang trở chia áp (quang trở lên 3V3, 10 kΩ xuống GND) | GPIO35 (ADC1_CH7) |
| OLED SSD1306 I2C 0x3C | SDA 32, SCL 33 |

Kit + testboard dùng chung với nhóm (GV cho phép). Đổi chân trong `include/config.h`. Mạch chia áp quang trở (LDR lên 3V3) dùng `LDR_INVERT 0`.

## Task

| Task | Core | Prio | Chu kỳ / sự kiện | Kỹ thuật |
|---|---|---|---|---|
| T1 led1 | 1 | 2 | 100 ms (T = 200 ms) | vTaskDelayUntil, pvParameters |
| T2 sensor | 0 | 2 | 500 ms | mutex, event group, xTaskNotify |
| T3 btn1 | 1 | 3 | quét 10 ms | chống dội, vTaskDelay |
| T4 btn2 | 1 | 3 | quét 10 ms | xSemaphoreGive (binary) |
| T5 led3 | 1 | 4 | chờ semaphore | xSemaphoreTake, critical section |
| T6 oled | 0 | 1 | 500 ms | mutex, critical section, xEventGroupWaitBits |
| T7 control | 0 | 2 | chờ queue set | queue lệnh + queue set, suspend/resume/prio, stack monitor |
| T8 alarm | 0 | 3 | chờ notification | xTaskNotifyWait |
| timer | daemon | – | 2 s | software timer → give semaphore cho Task 7 |

Mọi task gọi `xEventGroupSync` ở đầu nên cùng xuất phát. Log: mỗi task tự in, Serial được bảo vệ bằng recursive mutex `g_serialMutex`. Mỗi kỹ thuật đánh dấu `[K1]`..`[K14]` trong code, bảng tra ở đầu `src/main.cpp`.

## Chạy

1. VS Code + PlatformIO, mở thư mục này, chọn env `esp32dev`. Build ✓ → Upload → → Serial Monitor 115200.
2. Gõ lệnh trong Serial Monitor: `help`, `stat`, `light`, `give`, `s 1` / `r 1` (dừng / chạy lại LED1), `p 6 2` (đổi priority Task 6).
3. Không có kit: F1 → *Wokwi: Start Simulator* (`diagram.json`).

Ghi chú:
- Đường dẫn có dấu ("ĐỢT 2") làm ld trên Windows lỗi → `platformio.ini` build ra `C:\Users\<tên>\.pio_build\rtos_24119055`; `scripts/copy_fw.py` chép firmware về `.pio/wokwi/` cho Wokwi.
- Windows cần driver CP210x (Silicon Labs) để hiện cổng COM.
- Upload không vào: giữ BOOT, nhấn RST, thả BOOT rồi upload lại.

## Chạy giả lập QEMU (đã dùng để kiểm thử)

Env `qemu_esp32` (QEMU không có ADC nên Task 2 dùng tín hiệu giả `SIM_NO_ADC`; không nạp bản này lên kit):

```bash
pio run -e qemu_esp32
esptool.py --chip esp32 merge-bin -o flash.bin --fill-flash-size 4MB 0x1000 bootloader.bin 0x8000 partitions.bin 0xe000 boot_app0.bin 0x10000 firmware.bin
qemu-system-xtensa -machine esp32 -icount 3 -display none -serial stdio -drive file=flash.bin,if=mtd,format=raw
```

Kết quả: [`docs/qemu_log.txt`](docs/qemu_log.txt). LED3 sáng đúng 3000 ms; 3 lần `give` liên tiếp cho 2 lượt sáng (binary semaphore giữ tối đa 1 token); báo tối ở 22 %, hết tối ở 37 %; thống kê in đúng mỗi 10 s qua queue set; không log nào bị bỏ.

## Kiểm thử trên kit (điền khi chạy)

| # | Thao tác | Mong đợi | Kết quả |
|---|---|---|---|
| 1 | Cấp điện | LED1 nháy 5 lần/s, Serial có dòng `[T7] 8 task da qua diem hen` | |
| 2 | Nhấn BTN1 | LED2 đảo, OLED đổi ON/OFF | |
| 3 | Nhấn BTN2 | LED3 sáng 3 s, OLED đếm 3.0 → 0.0 → OFF | |
| 4 | Nhấn BTN2 3 lần nhanh | LED3 sáng 2 lượt; log "da co token" | |
| 5 | Che quang trở | % giảm; dưới 25 % OLED hiện [TOI] | |
| 6 | `s 1` rồi `r 1` | LED1 dừng rồi nháy lại | |
| 7 | `stat` sau 10 phút | stack còn dư, log bỏ = 0 | |

## Chạy thực nghiệm trên kit (16/11/2026)

Nạp qua COM5 (CP2102), Serial Monitor 115200. Đạt: khởi động 8 task + điểm hẹn, OLED 0x3C, BTN1 đảo LED2, BTN2 → semaphore → LED3 sáng đúng 3000 ms (token được giữ), che quang trở → 0 % → Task 8 báo tối rồi hết tối, heap ổn định, drop 0. Ảnh terminal và ảnh trích video ở `docs/evidence/`.

Video chạy thực nghiệm mạch thật: `demo/Demo_ThucNghiem_FreeRTOS_ESP32_16-11-2026.mp4` (ở thư mục gốc repo).
