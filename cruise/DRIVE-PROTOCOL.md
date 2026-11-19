# Перша калібрувальна поїздка з записом S14 / First calibration drive with the S14 recording

2026-11-19, агент RC. До брифу `docs/cruise-set-speed-brief.md`, фаза 1. / Agent RC, for the brief's Phase 1.

---

## Українською

### Головне правило

Водій лише керує автомобілем і круїзом кнопками на кермі, як завжди. **Меню Signal Probe натискає тільки пасажир.** Водій гальмує, коли вважає за потрібне, і ніколи не відкладає гальмування заради запису: таке натискання педалі теж корисні дані. Усі сценарії лише на вільній дорозі, у межах дозволеної швидкості, у сухий світлий день, без різких маневрів. Якщо обстановка змінилася, сценарій припиняємо, позначку можна не ставити.

### Що потрібно

- Signal Probe з цієї гілки встановлений (версію ставить ведучий); у меню є рядки «Почати запис S14» і «Набір для запису».
- Пасажир з аркушем нотаток (шаблон нижче) і годинником.
- Для калібрування дистанції: рулетка 30 м або лазерний далекомір і будь-який нерухомий предмет розміром з автомобіль (інший припаркований автомобіль); на дорозі нічого не потрібно, крім стовпчиків чи знаків уздовж узбіччя.
- Бажано: відеореєстратор, що бачить дорогу попереду, з годинником, виставленим за телефоном (щоб зіставити відео з позначками).

### Ємність запису

Запис S14 займає приблизно 9 КБ/с на диску; у квоту 15 МБ вміщається близько 30 хвилин (при вищих частотах сигналів — 15). Коли квоту вичерпано, **найстаріші файли видаляються**, навіть з попередньої поїздки. Тому: записуємо блоками (кожен блок — окремий запис «старт / стоп»), разом **не більше 25 хвилин за поїздку**, і після поїздки ведучий забирає файли з `data/rec` до наступної. Рядок запису показує «вільно … МБ»: якщо менше 3 МБ — зупинити запис.

### Перед виїздом (на стоянці, двигун працює)

1. Відкрити Signal Probe. «Набір для запису: S14» (якщо ні — натиснути цей рядок, доки не стане S14).
2. Натиснути «Почати запис S14». Рядок має показати «Запис S14 іде, 0:05…» і число зразків, що росте. Записати в нотатки: час, номер наступної позначки (він на рядку «Позначка N»).
3. Якщо через хвилину зразків менше кількох сотень або рядок показує «ЗУПИНЕНО: …» — записати текст, зупинити, повідомити ведучому. Поїздку можна продовжити без запису.

### Блок 0 — дистанція на стоянці (5 хв, нерухомо)

Мета: прив’язати сирі одиниці `gapX` до метрів, якщо камера бачить нерухомий автомобіль.

1. Поставити наш автомобіль прямо позаду іншого припаркованого автомобіля на 5 м (рулеткою: від переднього бампера до заднього бампера іншого).
2. Пасажир: «Позначка», записати «M?: 5 м». Почекати 20 с.
3. Те саме на 10 м, 15 м, 20 м, 30 м (від’їжджаючи назад повільно; кожна відстань — позначка і запис у нотатках).
4. Зупинити запис. (Якщо камера не бачить нерухомих автомобілів, дистанцію калібруємо в блоці 2.)

### Блок 1 — кроки заданої швидкості (10–12 хв, один запис)

Вільна рівна дорога, попереду нікого на 300+ м, дозволена швидкість не нижче 100 км/год. Круїз DCC (не обмежувач).

1. «Почати запис». Розігнатися, увімкнути круїз на 80 км/год. Почекати 20 с рівного руху.
2. Пасажир ставить позначку **в момент натискання** водієм кнопки й записує, що зроблено. Кожен крок — потім 15–20 с без дій:
   - +1 (один короткий дотик), −1;
   - +5 (п’ять дотиків поспіль), −5;
   - +10 (натиснути за фіксацію: до наступного десятка), −10;
   - +20 (два натискання за фіксацію), −20.
3. Повторити на 60 км/год (якщо дозволено) і на 100 км/год, якщо ситуація дозволяє. Великі кроки вниз (−20) — головне вимірювання: чи гальмує круїз, чи лише скидає газ. Під ухил кроки вниз не робимо в цьому блоці.
4. Записати в нотатках, чи кнопка діяла як очікувалося (1 км/год за дотик, до десятка за фіксацією) — це перевірка з брифу.
5. Якщо попереду з’явився автомобіль — водій діє як звичайно (гальмує або обганяє), пасажир ставить позначку і пише «перешкода».
6. Зупинити запис.

### Блок 2 — рівномірне слідування на двох-трьох дистанціях (8–10 хв, один запис)

Попереду автомобіль, що рухається рівно (вантажівка на трасі — найкраще). Водій підбирає задану швидкість круїзу кнопками так, щоб дистанція трималася.

1. «Почати запис». Дистанція «далеко»: приблизно 4 с (стовпчик, повз який пройшов передній автомобіль, ми проходимо через 4 с). Тримати 60–90 с. Позначка на початку і в кінці.
2. Дистанція «середня»: 3 с. 60–90 с. Позначки.
3. Дистанція «2 с» (не ближче!). 60–90 с. Позначки.
4. **Прив’язка дистанції до метрів** (у кожному відрізку, 3–5 разів): пасажир ставить позначку, коли **задній бампер переднього автомобіля** проходить стовпчик / знак, і ще одну, коли **наш передній бампер** проходить той самий стовпчик. Записати: «M7 / M8 стовпчик». Дистанція = швидкість × час між позначками (обидві позначки з однаковою реакцією пасажира, тож затримка реакції майже скорочується). Чим рівніша швидкість, тим точніше; при 90 км/год похибка 0,1 с — це 2,5 м, тому повторюємо кілька разів.
5. Зупинити запис.

### Блок 3 — плавне сповільнення переднього (5–10 хв, за нагоди)

Найкраще з другом на другому автомобілі як «переднім» (зв’язок через гучний зв’язок телефону, лише голос). Інакше — природні сповільнення потоку.

1. «Почати запис». Слідування з дистанцією 3 с на круїзі.
2. Передній **плавно** зменшує швидкість (наприклад, 90 → 70 км/год за 10–15 с). Пасажир ставить позначку на початку сповільнення (стоп-сигнали переднього).
3. Водій **спершу знижує задану швидкість кнопками** (−5, −10), як робив би алгоритм; якщо дистанція скорочується нижче 2 с — гальмує педаллю без вагань. Пасажир позначає кожне натискання кнопки й педалі гальма («M12 −10», «M13 гальмо»).
4. Різке гальмування переднього не імітуємо.
5. Зупинити запис.

### Після поїздки

- Зупинити останній запис (рядок знову «Почати запис S14»).
- Ведучий забирає папку `/var/opt/public/oap/oap_<uid>/data/rec/` і журнал `signal_probe.log` (через SSH, як для інших збирань) і конвертує: `python3 tools/car/rec_convert.py rec -o out --resample 100`.
- Нотатки пасажира — фото або текст ведучому.

### Шаблон нотаток пасажира

| Позначка | Час | Блок / сценарій | Що зроблено (кнопка, педаль, стовпчик) | Швидкість на дисплеї | Заданий круїз | Примітки (передній, ухил, погода) |
|---|---|---|---|---|---|---|
| M1 | 10:02:15 | 0 | 5 м рулеткою | 0 | — | |

### Як сирі одиниці дистанції прив’язуються до метрів

1. Для кожної пари позначок «передній пройшов стовпчик / ми пройшли стовпчик»: дистанція d = v · Δt, де v — наша швидкість у ці секунди (`car-speed2` × 1/64 км/год, перевірена на автомобілі шкала, ÷ 3,6), Δt — час між позначками із запису.
2. Сире `gapX` у ті самі секунди — з запису (`front-vehicle.frontVehicleGapX`), середнє за інтервал.
3. Пряма через точки (d, gapX) зі статичних (блок 0) і динамічних (блок 2) вимірювань дає масштаб і зсув; розкид точок — похибку. Значення 4093 / 4095 — «немає цілі».
4. Відносну швидкість (`frontVehicleSpeedX`, близько 1500 при рівних швидкостях) калібруємо після цього похідною відкаліброваної дистанції.

---

## In English

### The main rule

The driver only drives and operates the cruise with the steering-wheel buttons, as usual. **Only the passenger touches the Signal Probe menu.** The driver brakes whenever needed and never delays braking for the sake of the recording: those pedal presses are data too. All scenarios on an open road, within the speed limit, on a dry day in daylight, no harsh manoeuvres. If the traffic changes, the scenario stops; a missed mark does not matter.

### What is needed

- Signal Probe from this branch installed (the lead sets the version); the menu has "Почати запис S14" (start recording) and "Набір для запису" (recording set).
- The passenger with a notes sheet (template below) and a watch.
- For the gap calibration: a 30 m tape or a laser rangefinder and a parked car as the target; on the road only roadside posts or signs.
- Preferably a dash camera facing forward, its clock set from the phone (to match the video with the marks).

### Recording capacity

S14 writes about 9 KB/s to disk; the 15 MB quota holds about 30 minutes (15 at higher signal rates). When the quota is full **the oldest files are deleted**, even those of an earlier drive. So: record in blocks (each block its own start / stop), **at most 25 minutes per drive in total**, and the lead copies `data/rec` off before the next drive. The recording row shows "вільно … МБ" (free): below 3 MB, stop the recording.

### Before leaving (parked, engine running)

1. Open Signal Probe. "Набір для запису: S14" (if not, press that row until it says S14).
2. Press "Почати запис S14". The row must show "Запис S14 іде, 0:05…" and a growing sample count. Note the time and the next mark number (on the "Позначка N" row).
3. If after a minute there are fewer than a few hundred samples, or the row says "ЗУПИНЕНО: …" (stopped), note the text, stop, tell the lead. The drive can go on without a recording.

### Block 0 — gap while parked (5 min, standing)

Goal: tie the raw `gapX` to metres, if the camera reports a standing car.

1. Park straight behind another parked car at 5 m (tape: our front bumper to its rear bumper).
2. Passenger: "Позначка", note "M?: 5 m". Wait 20 s.
3. The same at 10, 15, 20, 30 m (rolling back slowly; a mark and a note at each).
4. Stop the recording. (If the camera does not report standing cars, the gap is calibrated in block 2.)

### Block 1 — set-speed steps (10–12 min, one recording)

An open, level road, nobody ahead for 300+ m, a limit of at least 100 km/h. Cruise in DCC (not the limiter).

1. Start the recording. Set the cruise at 80 km/h. 20 s of steady driving.
2. The passenger marks **the moment the driver presses** and notes what was done. After each step 15–20 s without input:
   - +1 (one short tap), −1;
   - +5 (five taps), −5;
   - +10 (press past the detent: to the next ten), −10;
   - +20 (two presses past the detent), −20.
3. Repeat at 60 km/h (where allowed) and at 100 km/h when safe. The large downward steps (−20) are the key measurement: does the cruise brake or only lift off? No downward steps on a downhill in this block.
4. Note whether the buttons behaved as expected (1 km/h a tap, the next ten past the detent): the brief's open question.
5. If a vehicle appears ahead, the driver acts as usual (brakes or overtakes); the passenger marks it and writes "obstacle".
6. Stop the recording.

### Block 2 — steady following at two or three gaps (8–10 min, one recording)

A vehicle ahead at a steady speed (a lorry on a highway is best). The driver adjusts the cruise set speed with the buttons to hold the gap.

1. Start the recording. "Far": about 4 s (the lead passes a post, we pass it 4 s later). Hold 60–90 s. Marks at the start and the end.
2. "Middle": 3 s. 60–90 s. Marks.
3. "2 s" (never closer). 60–90 s. Marks.
4. **Tying the gap to metres** (in each stretch, 3–5 times): the passenger marks when **the lead's rear bumper** passes a post or sign, and again when **our front bumper** passes the same post. Note "M7 / M8 post". Gap = speed × time between the marks (both presses carry the same reaction delay, so it nearly cancels). The steadier the speed the better; at 90 km/h 0.1 s is 2.5 m, hence the repeats.
5. Stop the recording.

### Block 3 — a gentle slowdown of the lead (5–10 min, when possible)

Best with a friend in a second car as the lead (talking over the phone's hands-free, voice only). Otherwise natural slowdowns of the traffic.

1. Start the recording. Following at 3 s on the cruise.
2. The lead slows **gently** (e.g. 90 → 70 km/h over 10–15 s). The passenger marks the start (the lead's brake lights).
3. The driver **first lowers the set speed with the buttons** (−5, −10), as the algorithm would; if the gap drops below 2 s, brakes with the pedal without hesitation. The passenger marks every button and brake press ("M12 −10", "M13 brake").
4. No hard braking of the lead is staged.
5. Stop the recording.

### After the drive

- Stop the last recording (the row says "Почати запис S14" again).
- The lead copies `/var/opt/public/oap/oap_<uid>/data/rec/` and `signal_probe.log` (SSH, as for the other collections) and converts: `python3 tools/car/rec_convert.py rec -o out --resample 100`.
- The passenger's notes go to the lead (a photo or text).

### The passenger's notes template

| Mark | Time | Block / scenario | What was done (button, pedal, post) | Displayed speed | Cruise set speed | Notes (lead, slope, weather) |
|---|---|---|---|---|---|---|
| M1 | 10:02:15 | 0 | 5 m by tape | 0 | — | |

### How the raw gap units are tied to metres

1. For each pair of marks "the lead passed the post / we passed the post": gap d = v · Δt, where v is our speed in those seconds (`car-speed2` × 1/64 km/h, the scale verified on the car, ÷ 3.6) and Δt the time between the marks in the recording.
2. The raw `gapX` in the same seconds from the recording (`front-vehicle.frontVehicleGapX`), averaged over the interval.
3. A straight line through the points (d, gapX) from the static (block 0) and dynamic (block 2) readings gives the scale and the offset; the scatter gives the error. 4093 / 4095 mean "no target".
4. The relative speed (`frontVehicleSpeedX`, about 1500 at equal speeds) is calibrated afterwards against the derivative of the calibrated gap.
