Заметки "на промокашке" по использованию SDR приемника RSP1A в Linux
====================================================================

## О чём речь?
Речь идёт про достаточно дешевый клон SDR приёмника RSP1A.

Информацию об оригинальном продукте можно получить у авторов
(SDRplay Ltd): https://www.sdrplay.com/rsp1a/

Стоимость оригинального приёмника на момент написания данных заметок
составлял около $109, что не дешево (когда-то мы за эти деньги покупали
и LimeSDR-mini и ADALM-Pluto, но это это уже приёмо-передатчики).

Из интересных особенностей RSP1A:
* АЦП 14 разрядов - очень неплохо в сравнении с 8 разрядами народного RTL-SDR,
на уровне большинства популярных SDR типа LimeSDR, ADALM Pluto, AirSpy, USRP.
* Приём от 1 кГц до 2 ГГц (но в "нашем" случае будет скорее до 1 ГГц).
* Полоса приёма до 10 МГц (но 14 бит АЦП только до 2 МГц).
* Стабильность опорного генератора (TCXO 24МГц) 0.5PPM (достойный уровень).
* В целом заявляются не плохие параметры чувствительности (для оригинального продукта).
* Оригинальный продукт содержит диапазоные переключаемые фильтры
(0-2МГц, 2-12МГц, 12-30МГц, 30-60МГц, 120-250МГц, 250-300МГц, 300-380МГц,
380-420мГц, 420-1000МГц, >1ГГц), малошумящий управляемый усилитель (LNA,
Gain Control).

Устройство основано на двух чипах:
* MSI001 - аналоговый приёмник;
* MSI2500 - АЦП, модуль DSP и USB контроллер.

На Aliexpress представлены некоторые упрощенные клоны данного SDR
приемника. Сейчас их цена за плату оставляет от 3 тыс. руб. РФ.
Некоторое время это было всего 1,5 тыс. руб., что практически приравнивает
стоимость этих устройств к "тюнинговым" версиям RTL-SDR продаваемых специально
для SDR энтузиастов (не путать с бестолковыми DVB-T донглами с бытовым, а не SMA
разъемом и ужасным опорным генератором).

Клон является несколько упрощенным устройством на котором имеются два основных
чипа, TCXO 0.5PPM, упрощённые (более "широкие") диапазонные фильтры не переключаемые
фильтры (для каждого диапазона выбирается свой входной SMA разъем), отсутствует
управляемый МШУ (LNA).

Подробно на gitgub имеется некоторый "реверс" этой платы:
https://github.com/EndlessEden/msiSDR/tree/RSP1-S

И, соответственно, есть Open Sorce (Open Hardware) дизайн подобной платы в KiCAD
https://github.com/EndlessEden/msiSDR/

Именно подобные платы нас и интересует в первую очередь за счёт малой цены,
простоты приобретения и, даже в некотором смысле, открытости железа.

Мы получаем 5 входом для 5 диапазонов: 0-30 МГц, 30-60 МГц, 50-120 МГц,
120-250 МГц, 400-1000 МГц.

Есть и другие варианты вариантов исполнения "клонов": 0-60 МГц, 30-60 МГц,
50-120 МГц, 120-250 МГц, 250-420, 400-1000 МГц

## Что с soft'ом?

Для Windows производитель предлагает использовать некоторый SDRuno в
качестве ПО. Это не наш вариант! Что есть под Linux и желательно Open Source?

Что опробовано:
- работает проприетарная библиотека API для sdrplay;
- вяло работает soapyMiri (врет по частоте);
- не работает miri (нужно разбираться);
- работает gqrx "из коробки" с sdrplay;
- как-то работает  CubicSDR.

Что не работает и это возмутительно:
- пока не удалось запустить Gnuradio с RSP1A (gnuradio-companion).

Есть библиотека от производителя "API", которая поддерживается для большинства
платформ. Библиотека противопригарная, но "на безрыбье и рак рыба".
Ссылка: https://www.sdrplay.com/api/
На момент паписания заметки это был файл `SDRplay_RSP_API-Linux-3.12.1.run`.

Для данного API имеется Open Sorce драйвер Soapy и Gnuradio, снимется
возможность запуска "готовых" приложений типа gqrx-sdr и CubicSDR, можно
использовать gnuradio-companion для прототипирования.
В настоящее время (у меня ядро 6.1.0) пока не поддерживается "как надо" данное
железо на уровне ядра (приходится отключать модули, чтобы работать с устройством,
через проиприетарную библиотеку от разработчика и lubusb).
Но обо всем по порядку.

Далее все действия производились в Debian-12 (вероятно в случае Ubuntu или Debian
других версий могут быть те или отличия).

### Устанавливаем зависимости
```bash
$ sudo apt install libusb-dev libudev-dev
$ sudo apt install git build-essential automake cmake
$ sudo apt install soapysdr-tools libsoapysdr-dev soapysdr-module-audio
$ sudo apt install gnuradio
```

### Блокируем загрузку модулей ядра, которые нам будут только мешать
```bash
sudo modprobe -r msi2500 msi001
```
Создаем файл `/etc/modprobe.d/blacklist-msi-rsp.conf` следующего содержания:
```
blacklist sdr_msi3101
blacklist msi001
blacklist msi2500
```
Это можно сделать с помощью echo или вашего любимого текстового редактора]
(vim/nano и т.п.).

### Установка проприетарной библиотеки API от SDRplay Limited.
- идем на https://www.sdrplay.com/downloads/
- скачиваем архив вида `SDRplay_RSP_API-Linux-3.12.1.run` (или другой актуальной версии)
- запускаем:
```
$ chmoad a+x ./SDRplay_RSP_API-Linux-3.12.1.run
$ ./SDRplay_RSP_API-Linux-3.12.1.run
```
- соглашаемся с лицензией (а кто их читает?)
- соглашаемся с установкой файлов в /opt/sdrplay_api, /usr/local/ и др.
- запускаем сервис `sdrplay`:
```
$ sudo systemctl enable sdrplay --now
```
Далее, в случае "глюков" (а они могут быть) запоминаем команду для перезапуска
сервиса после переподключения устройства:
```
$ sudo systemctl restart sdrplay
```
- посмотреть системный журнал сервиса можно традиционным образом:
```
$ sudo journalctl -u sdrplay -o cat -f
```

Далее "развилка" - для работы можно использовать несколько различных библиотек
следующего уровня: Soapy "miri" или "sdrplay" или "soapyMiri". Да!
Можно запутаться! Ну придется все по порядку

### Установка soapy 'miri' драйвера из пакетов Debian (ЭТО НЕ РАБОТАЕТ)
У меня этот драйвер кажется не работает "как надо", возможно нужно удалить
'sdrplay' драйвер, ещё не проверял.
Потому, вероятно можно пропустить данный раздел.
```bash
$ sudo apt install soapysdr-module-mirisdr
$ SoapySDRUtil --info
```
Проверить, что устройство "находится":
```bash
$ SoapySDRUtil --find
```
Правильный вывод должен содержать что-то вроде:
```
  Found device 2
  driver = miri
  label = Mirics MSi2500 default (e.g. VTX3D card)
  miri = 0
```
Проверить работы драйвера:
```bash
$ SoapySDRUtil --probe="driver=miri"
```
Вот тут я и получил сообщение об ошибке вида:
```
Lost samples!
OOOOOOOOOOOOOOOOOOO...
```
Повторюсь, возможна проблема связана с одновременной установкой sdrplay
драйвера, возможно стоит попробовать установить miri драйвер из исходников.

### Установка soapy 'sdrplay' драйвера из исходных текстов
Этот вариант мне показался рабочим, потому выполняем:
```bash
$ git clone https://github.com/pothosware/SoapySDRPlay.git
$ cd SoapySDRPlay
$ mkdir _build
$ cd _build
$ cmake .. -DCMAKE_BUILD_TYPE=Release
$ make -j 4
$ sudo make install
$ sudo ldconfig
$ SoapySDRUtil --info
```
Проверить, что устройство "находится":
```bash
$ SoapySDRUtil --find
```
Правильный вывод должен содержать что-то вроде:
```
Found device 3
  driver = sdrplay
  label = SDRplay Dev0 RSP1 0000000001
  serial = 0000000001
```
Проверить работы драйвера:
```bash
$ SoapySDRUtil --probe="driver=sdrplay"
```
У меня в отличии от "miri" эта операция выполнилась без ошибок! Ура!

### Установка soapy 'soapyMiri' драйвера из исходных текстов (РАБОТАЕТ КРИВО)
```bash
$ git clone https://github.com/ericek111/libmirisdr-5
$ cd libmirisdr-5
$ mkdir _build && cd _build
$ cmake ..
$ make -j4
$ sudo make install
$ sudo ldconfig
```
```bash
$ git clone https://github.com/ericek111/SoapyMiri
$ cd SoapyMiri
$ mkdir _build && cd _build
$ cmake ..
$ make -j4
$ sudo make install
$ sudo ldconfig
```
Проверить, что устройство "находится":
```bash
$ SoapySDRUtil --find
```
Правильный вывод должен содержать что-то вроде:
```
Found device 4
  driver = soapyMiri
  index = 0
  label = Mirics MSi2500 default (e.g. VTX3D card) :: 00000001
  manufacturer = Mirics
  product = MSi2500
  serial = 00000001
```
Проверить работы драйвера:
```bash
$ SoapySDRUtil --probe="driver=soapyMiri"
```
Операция должна завершиться без ошибок!

### Еще одна попытка установить `miri`
```bash
$ git clone https://github.com/f4exb/libmirisdr-4
$ cd libmirisdr-4
$ mkdir _build && cd _build
$ cmake ..
$ make -j4
$ sudo make install
```

### Установка "gqrx"
Я использовать gqrx из поставки Debian-12:
```bash
$ sudo apt install gqrx-sdr
```
Все заработало "из коробки".
При первом запуске нужно выбрать устройство (Device): "SDRplay DevX RSP1..." -
т.е. выбираем драйвер `sdrplay` с проприетарной API библиотекой.
С драйвером `soapyMiri` программа работает "как-то странно" (не работает,
как надо).
Программа иногда "глючит", чтоб "сбросить" все настройки нужно удалить каталог
с файлами конфигурации:
```
$ rm ~/.config/gqrx/
```
К этой программе у меня "особые" теплые чувства, много лет назад я добавил в нее
стереодекодер для WFM и мой патч принял разработчик. Позже кто-то добавил и
стереодекодер для российского (советского) УКВ диапазона (WFM orit), но это уже
был не я, но предполагаю, что наш соотечественник из USSR.

### Установка CubicSDR
Эта программа тоже работает, у меня не большой опыт ее использования,
но она тоже как-то работает, если выбрать sdrplay, а не miri.
```bash
$ sudo apt install cubicsdr
```

### Установка SDRangel
Смотрим тут:
https://github.com/f4exb/sdrangel/wiki/Compile-from-source-in-Linux
```bash
$ sudo apt install qtbase5-dev qtbase5-private-dev qtlocation5-dev \
  qtmultimedia5-dev qtpositioning5-dev qtwebengine5-dev libqt5charts5-dev \
  libqt5serialport5-dev libqt5texttospeech5-dev libqt5websockets5-dev \
  libqt5gamepad5-dev
$ sudo apt install ibserialdv-dev libfftw3-dev libboost-dev libopus-dev \
  libhackrf-dev libiio-dev libopengl-dev
```
```bash
$ git clone https://github.com/f4exb/sdrangel.git
$ cd sdrangel
$ mkdir _build && cd _build
$ cmake ..
$ make -j4
```
Пока не разбирался...

### Установка `SDRPlusPlus`
см. https://github.com/AlexandreRouma/SDRPlusPlus
...

### gr-sdrplay (НЕ ВЫШЛО)
```
$ git clone https://gitlab.com/HB9FXQ/gr-sdrplay
$ cd gr-sdrplay
$ mkdir _build && cd _build
$ cmake ..
```
И! Получаем ошибки.... Нужн оразбираться.
Тут надо понимать, что данный проект уже 5 лет не обновлялся!
Вероятно это тупиковая ветка развития ПО для RSP1 + Gnuradio....

### Ставим `gr-osmosdr-gqrx` - форк от создателя Gqrx (НЕ ВЫШЛО)
```bash
$ sudo aptitude install libosmosdr-dev
$ git clone https://github.com/csete/gr-osmosdr-gqrx
$ cd gr-osmosdr-gqrx
$ mkdir _build && cd _build
$ cmake ..
```
### Ставим gr-osmosdr с поддержкой sdrplay (НЕ ВЫШЛО)
Пока не понятно как... Исходники gq-osmosdr куда-то пропали
`-DENABLE_NONFREE=TRUE`

### Ставим `gr-soapy` (НЕ ВЫШЛО)
```
$ git clone https://github.com/dicta/gr-soapy
$ cd gr-soapy
$ mkdir _build && cd _build
$ cmake ..
$ make
```
Ошибки компиляции - опять заминка...

### Gnuradio companion (ПОКА НЕ РОАБОТАЕТ)
В gnuradio-companion можно использовать `gr-osmosdr`.

В "Device arguments" для блока "osmocom Source" нужно указать:
"sdrplay", но это пока НЕ работает!!
...
Нужно разбираться дальше...

## Полезные ссылки на источники информации:
- https://www.sdrplay.com/rsp1a/
- https://www.sdrplay.com/downloads/
- https://revspace.nl/Msi2500SDR
- https://github.com/pothosware/SoapySDR/wiki
- https://github.com/EndlessEden/msiSDR/
- https://github.com/pothosware/SoapySDRPlay3/wiki
- https://victor-sudakov.livejournal.com/478315.html
- https://aliexpress.ru/item/1005003654127606.html?gatewayAdapt=nld2rus&sku_id=12000027257897598
