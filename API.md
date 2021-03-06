
# #include <pms.h>

To use pms5003 library just install the library and `#include <pms.h>`

Nothing more. All described files are included automatically.

# Serial driver

To use another communication library:

* Implement interface `IPmsSerial` defined in the pmsSerial.h
* It defines kind of proxy between available serial communications libraries and pms5003 operations
  * My implementation of `IPmsSerial` for clone of AltSoftSerial is located in pmsSerialAltSoftSerial.h
* Modify pmsConfig.h
  * somewhere near `#define PMS_ALTSOFTSERIAL`
* Create object `yourDriver` from your implementation.
  * Use that object in constructor `Pms(&yourDriver);` or `.addSerial(&yourDriver);`

# pmsConfig.h

## #define PMS_DYNAMIC

Described in [Initialization C++ Style](README.md#initialization-c-style)

# tribool.h

It is a port of [boost.tribool](http://www.boost.org/doc/libs/1_63_0/doc/html/tribool.html) library - tristate logic: yes/no/unknown. It is used as a type of return value of `isModeActive()` and `isModeSleep()`. Sample code:
```C++
    if (pms.isModeActive()) {
        Serial.println("Active");
    } else if (!pms.isModeActive()) {
        Serial.println("Passive");
    } else {
        Serial.println("Unknown");
    }
```

Please note, that the goal of the code within `setup()` is to enter `isModeActive()` and `isModeSleep()` into any known state (true or false). Than (till the next `setSerial();`) state of `pms` is always known.
```C++
    if (!pms.write(pmsx::PmsCmd::CMD_RESET)) {
        pms.write(pmsx::PmsCmd::CMD_SLEEP);
        pms.write(pmsx::PmsCmd::CMD_WAKEUP);
    }
```

tribool.h defines both:
* third state
* tribool operators: `&&`, `||`, `==`, `!=`

More info: [myArduinoLibraries](https://github.com/jbanaszczyk/ArduinoLibraries)

# pms.h

## namespace pmsx

Almost all entities defined by <pms.h> are located in `pmsx::` namespace.

Please refer to [namespace pmsx{}](README.md#namespace-pmsx) description how to save typing _pmsx::_

## pmsxApiVersion

* `constexpr char pmsxApiVersion[];`
  * defines some kind of pms5003 API version

## PmsStatus

* `class PmsStatus`
  * represents status of [`Pms.read();`](#class-pms) operations. PmsStatus looks like enum class, but provides additional method `getErrorMsg();`
  * values:
    * `PmsStatus::OK`
    * `PmsStatus::NO_DATA`
    * `PmsStatus::READ_ERROR`
    * `PmsStatus::FRAME_LENGTH_MISMATCH`
    * `PmsStatus::SUM_ERROR`
    * `PmsStatus::NO_SERIAL`
  * `getErrorMsg();`
    * returns description of the status

## pmsData_t

* `typedef ... pmsData_t;`
  * type of data transmitted from PMS5003 sensor

## class PmsData

* `class PmsData`
  * Represents data received from PMS5003 sensor during [`Pms.read();`](#class-pms)
  * Provides some data types and constants appropriate for received data
  * `typedef ... pmsIdx_t;`
    * very handy type of iterator: `for (pmsx::PmsData::pmsIdx_t i = 0; i < view.getSize(); ++i) {`
  * `static constexpr pmsIdx_t DATA_SIZE;`
     * size (in terms of pmsData_t) of usable data received from sensor, probably useless in sketch
  * `static constexpr size_t RESPONSE_FRAME_SIZE;`
     * number of bytes in data frame received from sensor after write command, probably useless in sketch
  * `static constexpr size_t FRAME_SIZE;`
    * number of bytes in data frame, used as argument for [`Pms.waitForData();`](#class-pms), probably useless in sketch
  * `static constexpr size_t getFrameSize();`
    * same as `FRAME_SIZE` (C++ style), probably useless in sketch
  * `template <pmsIdx_t Size, pmsIdx_t Ofset> class PmsConcentrationData;`
    * template for "views": `concentrationCf`, `concentration`, `reserved`,  should never use it directly, just use `auto`: for example `auto view = data.concentration;`
    * `static constexpr pmsIdx_t getSize();`
      * returns size of the view (in terms of pmsData_t), iterate using `pmsIdx_t`
    * `pmsData_t getValue(pmsIdx_t index);`
      * returns single data value from the view
    * `static const char* getName(pmsIdx_t index);`
      * returns description of single data value from the view
    * `static const char* getMetric(pmsIdx_t index);`
      * returns metric name of single data value from the view
    * `static float getDiameter(pmsIdx_t index);`
      * returns particle diameter of single data value from the view
    * `[]`
      * same as `getValue()`, C style, note [Examples\p02cStyle\p02cStyle.ino](https://github.com/jbanaszczyk/pms5003/blob/master/Examples/p02cStyle/p02cStyle.ino)
    * `names[]`
      * same as `getName()`, C style, note [Examples\p02cStyle\p02cStyle.ino](https://github.com/jbanaszczyk/pms5003/blob/master/Examples/p02cStyle/p02cStyle.ino)
    * `metrics[]`
      * same as `getMetric()`, C style, note [Examples\p02cStyle\p02cStyle.ino](https://github.com/jbanaszczyk/pms5003/blob/master/Examples/p02cStyle/p02cStyle.ino)
    * `diameters[]`
      * same as `getDiameter()`, C style, note [Examples\p02cStyle\p02cStyle.ino](https://github.com/jbanaszczyk/pms5003/blob/master/Examples/p02cStyle/p02cStyle.ino)
  * `class Cleanliness;`
    * inherits from `template <...> class PmsConcentrationData;`, class is suitable for `particles` "view", should never be used directly, just `auto`: for example `auto view = data.particles;`
    * provides all public methods from `PmsConcentrationData<>` template described above
    * `float getLevel(pmsIdx_t index);`
      * returns cleanliness level of single data value from the view, according to "ISO 14644-1:2002"
  * view `PmsConcentrationData<...,...> raw;` (usage: `auto view = data.raw;`)
    * all data items received form PMS5003 sensor
  * view `PmsConcentrationData<...,...> concentrationCf;` (usage: `auto view = data.concentrationCf;`)
    * refers to PM___ concentration unit μg/m3 CF=1，standard particle (see [Appendix I](https://github.com/jbanaszczyk/pms5003/blob/master/doc/pms5003-manual_v2-3.pdf))
  * view `PmsConcentrationData<...,...> concentration;` (usage: `auto view = data.concentration;`)
    * refers to PM___ concentration unit μg/m3（under atmospheric environment) (see [Appendix I](https://github.com/jbanaszczyk/pms5003/blob/master/doc/pms5003-manual_v2-3.pdf))
  * view `Cleanliness particles;` (usage: `auto view = data.particles;`)
    * indicates the number of particles with diameter beyond ___ μm in 0.1 L of air (see [Appendix I](https://github.com/jbanaszczyk/pms5003/blob/master/doc/pms5003-manual_v2-3.pdf))
  * view `PmsConcentrationData<...,...> reserved;` (usage: `auto view = data.reserved;`)
    * probably useless

## class PmsCmd

* `enum class PmsCmd`
  * `PmsCmd` represents value to be sent by `write()` method. Commands are sent using serial communication or driving PIN3 and PIN6

|`PmsCmd`           | Medium                |
|-------------------|------------------------|
|`CMD_READ_DATA`    | Serial               |
|`CMD_MODE_PASSIVE` | Serial               |
|`CMD_MODE_ACTIVE`  | Serial               |
|`CMD_SLEEP`        | Serial or PIN 3 LOW   |
|`CMD_WAKEUP`       | Serial or PIN 3  HIGH |
|`CMD_RESET`        | PIN 6 LOW             |

## class Pms

* class `Pms` represents PMS5003 sensor operations
* class `Pms` constructors/destructor:
  * default constructor `Pms();`
    * does nothing (initializes fields representing state)
    * _Do It Yourself_: execute `addSerial(&pmsSerial);`
    * _Do It Yourself_: execute `begin();`
  * constructor `explicit Pms(IPmsSerial* pmsSerial);`
    * initializes fields representing state
    * executes `addSerial(pmsSerial);`
    * _(default)_ if symbol `PMS_DYNAMIC` is **not defined**: _Do It Yourself_: execute `begin();`
    * if symbol `PMS_DYNAMIC` is defined: executes `begin();`
  * destructor `~Pms();`
    * if symbol `PMS_DYNAMIC` is defined: executes `end();`
* class `Pms` provides a few usable constants:
  * `static constexpr unsigned long int TIMEOUT_PASSIVE = 68U;`
    * it is used as serial port driver default timeout
    * Calculation: transfer time of 1start + 32data + 1stop using 9600bps is 33 milliseconds. TIMEOUT_PASSIVE could be at least 34, Value of 68 is arbitrary doubled
  * `static constexpr unsigned int WAKEUP_TIME = 2500U;`
    * used in `waitForData()` after `write()` command with `CMD_WAKEUP` or `CMD_RESET`
    * established experimentally
* more class `Pms` initialization methods:
  * `void addSerial(IPmsSerial* pmsSerial);`
    * Just defines serial port driver to be used during communication, nothing more
  * `bool begin();`
    * initializes communication between PMS5003 and a serial driver
    * if serial driver' `begin()` returns `true`: `Pms::begin()` returns `true` too
    * otherwise it removes serial driver (`addSerial(nullptr);`) and returns `false`
  * `void end()`
    * stops communication between serial driver and PMS5003 (just `pmsSerial->end();`)
  * `void setTimeout(unsigned long int);`
    * modifies timeout for serial port communication. Default timeout is set to `TIMEOUT_PASSIVE` (68 milliseconds)
* class `Pms` provides information about sensor state:
  * `tribool isModeActive();`
    * state changes:
      * yes: `CMD_RESET`, `CMD_MODE_ACTIVE` from `!isModeActive()`, `CMD_WAKEUP` from `isModeSleep()`
      * no: `CMD_MODE_PASSIVE` from `isModeActive()`
      * unknown: initially
  * `tribool isModeSleep();`
    * state changes:
      * yes: `CMD_SLEEP` from `!isModeSleep()`
      * no: `CMD_RESET`, `CMD_WAKEUP` from `isModeSleep()`
      * unknown: initially
  * `bool isWorking();`
    * [#18](/../../issues/18) in future releases will be refactored to [PmsStatus](#class-pmsstatus)
    * returns true if sensor properly transmits (`read()` and `write()`) data
  * `bool initialized();`
    * returns true if there is assigned a serial driver
  * `unsigned long int getTimeout(void);`
    * returns current serial port operations timeout
      * initially: TIMEOUT_PASSIVE
      * can be changed by `setTimeout()`
* class `Pms` allows assigning PMS5003 pins 3 and 6 (SET and RESET) to Arduino pins
  * `void setPinSleepMode(uint8_t pinNumber);`
    * assigns PMS5003 PIN3 to Arduino PIN pinNumber
      * no test is made after assignment
      * it is OK to leave PMS5003 PIN3 not connected. Sleep/WakeUp command will be sent using serial communication
      * previously assigned pin (if any) is left in `OUTPUT` `HIGH` state
  * `void setPinReset(uint8_t pinNumber);`
    * assigns PMS5003 PIN6 to Arduino PIN pinNumber
      * no test is made after assignment
      * it is OK to leave PMS5003 PIN6 not connected. Reset command can be emulated sending `CMD_SLEEP` + `CMD_WAKEUP` sequence
      * previously assigned pin (if any) is left in `OUTPUT` `HIGH` state
  * `static constexpr auto pinNone;`
    * value can be used to unassign previously assigned hardware pins (SleepMode, Reset)
      * example: `pms.setPinSleepMode(pmsx::Pms::pinNone);`
      * example: `pms.setPinReset(pms.pinNone);`
* class `Pms` allows simple monitoring of serial driver activity
  * `size_t available();`
    * returns number of bytes sent by PMS5003 but not read yet
    * `available()` removes data from serial port buffer if they looks as a garbage
      * the first byte in the buffer should be `0x42` - the first byte of properly formatted data frame
  * `bool waitForData(unsigned int maxTime);`
    * waits
      - not longer than maxTime
      - wait for **any** data available from serial driver
      - data can be even garbage
      - returns `true` if there are ready any data in serial port buffer
  * `bool waitForData(unsigned int maxTime, size_t nData );`, `nData > 0`
    * waits
      - not longer than maxTime
      - wait for **looking good** data from serial driver
      - garbage is removed using `available()` described above
      - to wait for the whole, well formatted frame: `pms.waitForData(pmsx::Pms::TIMEOUT_PASSIVE, pmsx::PmsData::FRAME_SIZE);`
      - returns true if there are waiting `nData` bytes in the serial port buffer
      - even if it returns `true` there is no guarantee that waiting frame is not corrupted
* class `Pms` allows driving PMS5003 sensor
  * `bool write(PmsCmd cmd);`
    * sends `cmd` of type [PmsCmd](#class-pmscmd) to the sensor.
    * after `CMD_WAKEUP` and `CMD_RESET` write  waits: `waitForData(TIMEOUT_PASSIVE, PmsData::RESPONSE_FRAME_SIZE);`
    * [#16](/../../issues/16) type of returned value will refactored to `PmsStatus` in future releases
  * `bool write(PmsCmd cmd, unsigned int wakeupTime);`
    * sends `cmd` of type [PmsCmd](#class-pmscmd) to the sensor.
    * after `CMD_WAKEUP` and `CMD_RESET` write  waits: `waitForData(wakeupTime, PmsData::RESPONSE_FRAME_SIZE);`
      * `wakeupTime` can be even `0`, be careful - PMS5003 may not respond to your commands
    * [#16](/../../issues/16) type of returned value will refactored to `PmsStatus` in future releases
* **class `Pms` reads data from PMS5003 sensor**
  * `PmsStatus read(PmsData& data);`
    * reads whole data frame into `data` of type [PmsData](#class-pmsdata)
    * returns status of type [PmsStatus](#class-pmsstatus)
    * does not block

## Notice about PMS5003 modes

### Passive mode

In passive mode sensor waits for `CMD_READ_DATA` command, then sends its data. Don't rely on that naive loop:
1. write(`CMD_READ_DATA`)
2. wait for data
3. read the data
4. and again ...

Do not rely on such naive loop: in case of any communication error Arduino will "wait for data" forever. Both writing command and reading input have to be independent.

### CMD_RESET vs. CMD_WAKEUP

It looks, that sequence: `CMD_SLEEP`+`CMD_WAKEUP` is identical to `CMD_RESET`

### CMD_RESET, CMD_WAKEUP

1. After reset (or wakeup) PMS5003 needs some time to start working. Sensor is blind, it does not respond to commands, please wait `pmsx::Pms::TIMEOUT_PASSIVE`
2. Then communication works fine, but sensor sends zeros instead of valuable data. It takes about 2.5 seconds
3. It is documented, that sensor requires about 30 seconds after reset to send accurate data.
4. It does not matter if sensor was active or passive state before `CMD_SLEEP`, it wakes up in active mode.

### CMD_SLEEP

Example of sleep procedure:
1. `write(CMD_SLEEP);`
2. `end();`
3. optionally: unassign serial driver: `setSerial(nullptr);`
4. turn off serial driver

Example of wakeup procedure:
1. turn on serial driver
2. optionally: reassign serial driver to PMS5003: `setSerial(&pmsSerial);`
3. `begin();`
4. `write(CMD_WAKEUP);`
5. `waitForData(pmsx::Pms::TIMEOUT_PASSIVE, pmsx::PmsData::FRAME_SIZE);`

### States

There is the simplified summary to PMS5003 states. In fact there are more states, for example: awakening not responding for commands, awakening sending zeros, etc.


| States                                              | Output                                       |
|-----------------------------------------------------|----------------------------------------------|
| Sleeping                                            | None (waits for `CMD_WAKEUP` or `CMD_RESET`) |
| Active (awoken)                                     | Spontaneously sends data frames              |
| PassiveWait (awoken,waits for CMD_READ_DATA)        | None (waits for `CMD_READ_DATA`)             |
| PassiveSend (awoken,sends data after CMD_READ_DATA) | Sends single data frame                      |
