# Repairing the USB stick of the Breville / PolyScience / Sage "Control Freak" induction cooktop (and BORK U891)

**TL;DR:** the bundled USB stick shows up as `NAND USB2DISK`, **VID FFFF / PID 1201**, 0 bytes, "No Media".
It is a **FirstChip FC2279** controller + **SanDisk SDUNCIAMA-064GB** NAND whose firmware failed to load.
It **can be revived** with **FirstChip MpTools 2022-06-01** using a **slow NAND R/W cycle (100 ns)**,
**Quality** mode and a **fixed capacity of 7680 MB**. Tested 26.11.2026: survives replug and reboot, works in the cooktop.

Русская версия — [ниже](#ru).

---

## Which cooktops

- **Breville Commercial / PolyScience "The Control °Freak"** (CMC850, CMC850XL), sold in Europe as **Sage Polyscience**.
- **BORK U891** — the owner of this repo has a BORK; it is the same type of cooktop with the same special USB stick
  (rubber ring for wet hands, long neck so it reaches the recessed USB port). Ordinary flash drives do not reach the port.
- Breville spare part: **SP0011315 "USB"** (capacity not published anywhere).

The stick only stores your custom temperature programs (FAT32; the cooktop does not read NTFS). The cooktop does not
need it to work and it holds no cooktop firmware.

## Symptoms

- Mac / Windows / the cooktop do not see the stick, or the cooktop says it is "not formatted".
- Windows Device Manager: `USB Mass Storage Device`, `USB\VID_FFFF&PID_1201`, disk `NAND USB2DISK USB Device`,
  Disk Management: **No Media, 0 bytes**. ChipGenius: Controller **FirstChip FC1179/FC2279**.

VID FFFF / PID 1201 = FirstChip controller in **ROM (boot-loader) mode**: it cannot load its firmware from NAND.
In our case the NAND was fine, but the controller ↔ NAND link produced **bit errors at default speed**
(older tools read the flash ID as `413A98037268` instead of the real `453E98037664` — one bit flipped in every byte).
That is why it fails: corrupted tables at boot, huge "bad" areas, firmware not loading after power-cycle.

## What you need

- A **Windows** PC (the tools are Windows-only), admin rights.
- **FirstChip MpTools 2022-06-01** (MP Ver **1.0.5.2**). Archive names seen in the wild:
  `FirstChip_MpTools_20220601.rar` (usbdev.ru, FirstChip FC1178/FC1179 MpTools page) or
  `FirstChip.MpTools.v20220601.rar` (contains `(1234)FirstChip.MpTools.v20220601.7z`, password **1234**).
  Our copy for reference:
  - `FirstChip.MpTools.v20220601.rar` SHA-256 `d5e2bc352bef572ac8c13cc1197dd453ded968485f574321c3fc7cdb0d5bd7ef`
  - `FCMpTools.exe` SHA-256 `79592fce9c3e16708cc38b99a1cdffc042283b0732105c254b9ed7d2eee15008`
  These are third-party Chinese factory tools — scan them before running. We do not redistribute them.
- **Everything on the stick will be erased.** If you need the data, do not reflash — go to a data-recovery lab.

## Working recipe

1. Insert the stick, run `FCMpTools.exe` **as administrator**.
2. *Product Type* → **End Product** → OK → confirm "Yes".
3. *Language* → English. Click **Setting**, password is **empty** → OK.
4. Tab **Another Setting** → tick **Specify Flash r/w cycle**.
5. Tab **Scan**:
   - tick **Use public r/w cycle**, set **R/W Cycle = 100 ns** (the slowest);
   - Flash name: auto-detected as `453E9803766C SDUNCIAMA-064GB(9T25)` (leave "Spec Flash name" unticked);
   - **Scan Level = 4 – Clear + Factory scan**, ECC 12;
   - **Optimize = Quality**.
6. Tab **Bin** → tick **Fix capacity** → **All fix 7680 – 7680 MB**.
   (In these edit boxes Ctrl+A does not work: click, End, Shift+Home, Delete, then type.)
7. OK → **Refresh** → **Start**. Takes ~1.5 min → `Bin5, 11438MB=>7680MB, Succeed`.
8. **Right after flashing Windows may still show 0 bytes or hang — ignore it.** Unplug the stick (or reboot the PC),
   plug it back. Result: `USB\VID_346D&PID_5678`, drive **FAT32, ~8 GB**. Verify with a write/read test (H2testw etc.).
9. Do **not** enable *Auto Start / Auto Refresh* in MpTools — it re-flashes the stick every time it is detected.

How to tell the state: **VID 346D / PID 5678** = the stick booted its own firmware (good);
**VID FFFF / PID 1201** after a fresh plug-in = ROM mode (broken). Right after flashing, while the tool still holds the
stick, it may still show FFFF — check after replugging.

## What did NOT work (save yourself the time)

| Tool / settings | Result |
|---|---|
| MpTools **2024-02-21** (V1.0.7.2) | reads the ID correctly, but "电压设定错误" (voltage setting error) → **error code 23** on Start; no setting changes it |
| MpTools 2022-06-01, defaults | `UNKNOWN FLASH ID=413A98037268` |
| MpTools **2021-10-24** + forced flash type | "Succeed", 4 GB, 1 hour — but works **only while the tool holds it**; after replug → ROM mode again (2021 firmware does not boot on FC2279BA) |
| 2021-10-24 + 100 ns + Scan 4 | 16 GB in 1.5 min — same: does not boot by itself |
| 2022-06-01 + 100 ns + Scan 4 + **Capacity** mode, no fixed capacity | 15 GB, boots (346D) but after replug "No Media" |

The cooktop does **not** damage the stick — it failed on a PC alone. If the replacement still breaks, try a smaller
fixed capacity (e.g. 3840 MB) and keep 100 ns + Quality.

## Notes for AI agents (Claude Code etc.)

- Identify by `VID_FFFF&PID_1201` + `NAND USB2DISK` + ChipGenius "FirstChip FC1179/FC2279"; flash ID `453E98037664` /
  `453E9803766C` (SanDisk, 1CE TLC, 64 GB).
- Go straight to MpTools **2022-06-01** with the recipe above. Don't waste time on the 2024 build (error 23 by design for this
  combo) or the 2021 build (firmware does not boot).
- The key levers are **R/W cycle 100 ns** (fixes bit errors), **Optimize = Quality**, **fixed capacity 7680 MB**.
- Judge success only after a real power-cycle (unplug/replug or reboot), by VID 346D and a non-zero FAT32 volume.
- Useful references: usbdev.ru forum, FirstChip section (topic 4634 — same SanDisk 453E98… memory);
  liangchanba.com thread-1542 (FC1179/FC2279/FC212W tools and error codes).

---

<a id="ru"></a>
## По-русски

**Флешка от индукционной плиты Breville / PolyScience / Sage «Control Freak» и BORK U891** (прорезиненное кольцо,
длинный «нос»). Сломанная видна как `NAND USB2DISK`, **VID FFFF / PID 1201**, 0 байт, «нет носителя».
Внутри контроллер **FirstChip FC2279** и память **SanDisk 64 ГБ**. Контроллер в аварийном режиме: не может загрузить
прошивку из памяти, потому что обмен с памятью на обычной скорости идёт с ошибками.

**Лечится** программой **FirstChip MpTools 2022-06-01** (только Windows, всё на флешке сотрётся):
1. Запуск от администратора → End Product → «Да» → English → Setting (пароль пустой).
2. Another Setting → ✓ Specify Flash r/w cycle.
3. Scan → ✓ Use public r/w cycle, **R/W Cycle = 100 ns**; **Scan Level 4 (Clear + Factory scan)**; ECC 12;
   **Optimize = Quality**.
4. Bin → ✓ Fix capacity → **All fix 7680 – 7680 MB**.
5. OK → Refresh → Start (~1,5 мин, «Succeed»).
6. Сразу после прошивки может показывать 0 байт — **вынуть и вставить** (или перезагрузить ПК). Итог: VID 346D:5678,
   FAT32 ~8 ГБ, работает в плите.

**Не тратьте время:** версия 2024-02-21 даёт ошибку 23 «电压设定错误» (напряжение); версия 2021-10-24 «прошивает»,
но флешка живёт только до переподключения. Плита флешку не ломает. Подробности — в английской части выше.
