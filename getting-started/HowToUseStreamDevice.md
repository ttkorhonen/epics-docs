# How to use StreamDevice

W. Eric Norum <wenorum@lbl.gov>

## 1 - Introduction


This tutorial provides step-by-step instructions on how to create EPICS support for a simple serial, GPIB (IEEE-488) or network attached device.
The steps are presented in a way that should make it possible to apply them in cookbook fashion to create support for other devices.
For comprehensive description of all the details of the I/O system used here,
refer to the
[asynDriver](http://epics-modules.github.io/asyn/index.html) and
[StreamDevice](http://epics.web.psi.ch/software/streamdevice/doc/) documentation.

This document isn't for the absolute newcomer though.
You must have EPICS installed on a system somewhere
and know how to build
and run the example application.
In particular you must have the following installed:

*   EPICS R3.14.11 or higher.
*   EPICS ASYN support module version 4.14 or higher.
*   EPICS StreamDevice support module version 2.4.1 or higher.

Serial, GPIB
and network attached devices can now be treated in much the same way.
The EPICS StreamDevice and
ASYN support modules can use drivers which communicate with serial devices connected to ports on the IOC
or to Ethernet/Serial converters
or with GPIB devices connected to local I/O cards
or to Ethernet/GPIB converters
or to network attached devices using VXI-11
or plain TCP ('telnet') communication protocols.

I based this tutorial on the device support I wrote for a Hewlett Packard E3631A power supply.
I chose the E3610A as the basis for this tutorial partly because it was what I found lying around but mostly because it uses SCPI-compliant commands which are common to a wide range of devices.
Sections 1 through 4 of this tutorial apply to all SCPI (IEEE-488.2) devices.

## 2 - Create a new device support module

The easiest way to do this is with the `makeSupport.pl` script supplied with the EPICS ASYN package.

Here are the commands I ran.
To make the command examples a little shorter I used shell variables to hold the paths to my installation of EPICS base
and the ASYN support module.
You can do this with values appropriate for your site
or just type the full path names to the commands.

```makefile
EPICS_BASE=/usr/local/epics/R3.14.11/base
ASYN=/usr/local/epics/R3.14.11/modules/soft/synApps_5_5/support/asyn-4-14
```

You will have to change these to the values appropriate for your EPICS installation
or simply type the appropriate full paths to invoke the scripts.
You must also have the `EPICS_HOST_ARCH` environment variable set to match the machine on which you are building.

To create a new StreamDevice support module:

```console
norume> mkdir HPE3631A
norume> cd HPE3631A
norume> $ASYN/bin/$EPICS_HOST_ARCH/makeSupport.pl -t streamSCPI HPE3631A
```

### 2.1 - Make changes to some files in `configure/`

Edit the `configure/RELEASE` file which `makeSupport.pl` created
and confirm that the entries describing the paths to your EPICS base
and ASYN support are correct.
For example these might be:

```makefile
ASYN = /usr/local/epics/R3.14.11/modules/soft/synApps_5_5/support/asyn-4-14
EPICS_BASE = /usr/local/epics/R3.14.11/base
```

You'll also have to add a line describing the path to your StreamDevice support.
For example:

```makefile
STREAM = /usr/local/epics/R3.14.11/modules/soft/synApps_5_5/support/stream-2-4-1
```

Edit the `configure/CONFIG_SITE` file which `makeSupport.pl` created
and specify the IOC architectures on which the application is to run.
I wanted the application to run as a soft IOC,
so I uncommented the `CROSS_COMPILER_TARGET_ARCHS` definition
and set the definition to be empty:

```makefile
CROSS_COMPILER_TARGET_ARCHS =
```

You may have to fix an error in the `HPE3631ASup/devHPE3631A.db` file produced by the `makeSupport.pl` command.
Verify that the `IDNwf` record description looks like:

```
record(waveform, "$(P)$(R)IDNwf")
{
    field(DESC, "SCPI identification string")
    field(DTYP, "stream")
    field(INP,  "@devHPE3631A.proto getIDN(199) $(PORT) $(A)")
    field(PINI, "YES")
    field(FTVL, "CHAR")
    field(NELM, "200")
}
```

Some versions of the template have `FTYP` rather than `FTVL` for the waveform data type field.

I install the StreamDevice protocol file into the support module `db/` directory.
This seems like a reasonable place to put this file since it's tied closely to its associated database file.
To have the protocol file installed into the `db/` directory edit `HPE3631ASup/Makefile`
and add the line

```makefile
DB_INSTALLS += $(TOP)/HPE3631ASup/devHPE3631A.proto
```

to the list of `DB_INSTALLS`.

## 3 - Create a test application (optional)

Now that a device support module has been produced it is a good idea to create a new EPICS application to confirm that the device support is operating correctly.
The easiest way to do this is with the `makeBaseApp.pl` script supplied with EPICS.
The files in the `configure/` directory produced in the previous step do not provide all the information needed to build an application but will not be overwritten by the `makeBaseApp.pl` script.
The easiest solution to this problem is to simply remove the `configure/` directory
and all its contents
and
then to make the changes to `configure/RELEASE`
and `configure/CONFIG_SITE` at mentioned above.

Here are the commands I ran.


```console
norume> rm -rf configure
norume> $EPICS_BASE/bin/$EPICS_HOST_ARCH/makeBaseApp.pl -t ioc HPE3631Atest
norume> $EPICS_BASE/bin/$EPICS_HOST_ARCH/makeBaseApp.pl -t ioc -i HPE3631Atest
The following target architectures are available in base:
    darwin-x86
    RTEMS-mvme2100
    RTEMS-mvme3100
    RTEMS-uC5282
What architecture do you want to use? darwin-x86
The following applications are available:
    HPE3631Atest
What application should the IOC(s) boot?
The default uses the IOC's name, even if not listed above.
Application name?
norume>
```

I then added the ASYN
and STREAM values to `configure/RELEASE`
and made the change to the `CROSS_COMPILER_TARGET_ARCHS` line in `configure/CONFIG_SITE`.

You must add the StreamDevice
and ASYN support modules to the test application.
Edit `HPE3631AtestApp/src/Makefile`
and add the lines:


```makefile
HPE3631Atest_DBD += stream.dbd
HPE3631Atest_DBD += asyn.dbd
HPE3631Atest_DBD += drvAsynSerialPort.dbd
```
and
```makefile
HPE3631Atest_LIBS += stream asyn
```

To install the support module files
and build the test application simply run `make`:

```console
norume> make
```

The IOC startup script produced by the `makeBaseApp.pl` script must be modified before the test application can be run.
Change directories to `iocBoot/iocHPE3631Atest`
and edit the `st.cmd` file.
Here's the file that I use:


```bash
  1 #!../../bin/darwin-x86/HPE3631Atest
  2
  3 ###############################################################################
  4 # Set up environment
  5 < envPaths
  6 epicsEnvSet "STREAM_PROTOCOL_PATH" "$(TOP)/db"
  7

  8 ###############################################################################
  9 # Allow PV name prefixes and serial port name to be set from the environment
 10 epicsEnvSet "P" "$(P=hpE3631A)"
 11 epicsEnvSet "R" "$(R=Test)"
 12 epicsEnvSet "TTY" "$(TTY=/dev/tty.PL2303-000013FA)"
 13

 14 ###############################################################################
 15 ## Register all support components
 16 cd "${TOP}"
 17 dbLoadDatabase "dbd/HPE3631Atest.dbd"
 18 HPE3631Atest_registerRecordDeviceDriver pdbbase
 19

 20 ###############################################################################
 21 # Set up ASYN ports
 22 # drvAsynSerialPortConfigure port ipInfo priority noAutoconnect noProcessEos
 23 drvAsynSerialPortConfigure("L0","$(TTY)",0,0,0)
 24 asynSetOption("L0", -1, "baud", "9600")
 25 asynSetOption("L0", -1, "bits", "8")
 26 asynSetOption("L0", -1, "parity", "none")
 27 asynSetOption("L0", -1, "stop", "2")
 28 asynSetOption("L0", -1, "clocal", "Y")
 29 asynSetOption("L0", -1, "crtscts", "Y")
 30 asynOctetSetInputEos("L0", -1, "\n")
 31 asynOctetSetOutputEos("L0", -1, "\n")
 32 asynSetTraceIOMask("L0",-1,0x2)
 33 asynSetTraceMask("L0",-1,0x9)
 34

 35 ###############################################################################
 36 ## Load record instances
 37 dbLoadRecords("db/devHPE3631A.db","P=$(P),R=$(R),PORT=L0,A=0")
 38

 39 ###############################################################################
 40 ## Start EPICS
 41 cd "${TOP}/iocBoot/${IOC}"
 42 iocInit
```

Lines that you'll likely have to modify are:

|  |  |
|---|---|
| 10,11 | The prefix applied to all PV names. |
| 12 | The name of your serial line (for network attached devices or LAN/Serial adapters this will be something like 192.168.0.23:4001). |
| 23 | The command would be `drvAsynIPPortConfigure` for LAN/Serial adapters or network attached devices. See the asynDriver documentation for information on connecting to GPIB devices. |
| 24-29 | The serial line settings for your device (for network attached devices or LAN/Serial adapters or GPIB devices these lines can be removed or commented out). |
| 30,31 | GPIB devices may not need line terminators since they can use the EOI line to mark the end of messages. |
| 33 | This trace mask value prints a message showing the characters sent to and received from the device. This can produce a lot of messages on the IOC console. To display error messages only set the trace mask to 1 rather than 9. |

So, if you have a LAN/Serial adapter
or network attached device which uses a raw TCP
('telnet' style)
connection you would replace lines 23 through 29 with a single line like:

```bash
drvAsynIPPortConfigure("L0","192.168.0.23:4001",0,0,0)
```

You would also change the application Makefile to specify `drvAsynIPPort.dbd` rather than `drvAsynSerialPort.dbd`.

I you have a VXI-11 LAN/GPIB adapter
or network attached device you would replace lines 23 through 31 with a single line like:

```bash
vxi11Configure("L0","192.168.0.24",0,0.0,"gpib0",0,0)
```

You would also change the application Makefile to specify `drfvVxi11.dbd` rather than `drvAsynSerialPort.dbd`.

### 3.1 - Run the test application

The application can be started by running the startup script if you make the script executable:

```console
norume> cd iocBoot/iocHPE3631Atest
norume> chmod +x st.cmd
```

Then the application can be run by:
```bash
norume> ./st.cmd
#!../../bin/darwin-x86/HPE3631Atest
###############################################################################
# Set up environment
< envPaths
epicsEnvSet("ARCH","darwin-x86")
epicsEnvSet("IOC","iocHPE3631Atest")
epicsEnvSet("TOP","/Users/wenorum/tmp/t/HPE3631A")
epicsEnvSet("ASYN","/usr/local/epics/R3.14.11/modules/soft/synApps_5_5/support/asyn-4-14")
epicsEnvSet("STREAM","/usr/local/epics/R3.14.11/modules/soft/synApps_5_5/support/stream-2-4-1")
epicsEnvSet("EPICS_BASE","/usr/local/epics/R3.14.11/base")
epicsEnvSet "STREAM_PROTOCOL_PATH" "/Users/wenorum/tmp/t/HPE3631A/db"
###############################################################################
# Allow PV name prefixes and serial port name to be set from the environment
epicsEnvSet "P" "hpE3631A"
epicsEnvSet "R" "Test"
epicsEnvSet "TTY" "/dev/tty.PL2303-000013FA"
###############################################################################
## Register all support components
cd "/Users/wenorum/tmp/t/HPE3631A"
dbLoadDatabase "dbd/HPE3631Atest.dbd"
HPE3631Atest_registerRecordDeviceDriver pdbbase
###############################################################################
# Set up ASYN ports
# drvAsynSerialPortConfigure port ipInfo priority noAutoconnect noProcessEos
drvAsynSerialPortConfigure("L0","/dev/tty.PL2303-000013FA",0,0,0)
asynSetOption("L0", -1, "baud", "9600")
asynSetOption("L0", -1, "bits", "8")
asynSetOption("L0", -1, "parity", "none")
asynSetOption("L0", -1, "stop", "2")
asynSetOption("L0", -1, "clocal", "Y")
asynSetOption("L0", -1, "crtscts", "Y")
asynOctetSetInputEos("L0", -1, "\n")
asynOctetSetOutputEos("L0", -1, "\n")
asynSetTraceIOMask("L0",-1,0x2)
asynSetTraceMask("L0",-1,0x9)
###############################################################################
## Load record instances
dbLoadRecords("db/devHPE3631A.db","P=hpE3631A,R=Test,PORT=L0,A=0")
###############################################################################
## Start EPICS
cd "/Users/wenorum/tmp/t/HPE3631A/iocBoot/iocHPE3631Atest"
iocInit
Starting iocInit
############################################################################
## EPICS R3.14.11 $R3-14-11$ $2009/08/28 18:47:36$
## EPICS Base built Aug 24 2010
############################################################################
2010/08/31 13:10:32.986 /dev/tty.PL2303-000013FA write 6
\*IDN?\\n
iocRun: All initialization complete
2010/08/31 13:10:33.116 /dev/tty.PL2303-000013FA read 1
H
2010/08/31 13:10:33.135 /dev/tty.PL2303-000013FA read 1
E
2010/08/31 13:10:33.153 /dev/tty.PL2303-000013FA read 1
W
2010/08/31 13:10:33.171 /dev/tty.PL2303-000013FA read 1
L
2010/08/31 13:10:33.189 /dev/tty.PL2303-000013FA read 1
E
2010/08/31 13:10:33.208 /dev/tty.PL2303-000013FA read 1
T
2010/08/31 13:10:33.226 /dev/tty.PL2303-000013FA read 1
T
epics>dbpr hpE3631ATestIDN
ASG:
DESC: SCPI identification string
DISA: 0
DISP: 0
DISV: 1
NAME: hpE3631ATestIDN
SEVR: NO_ALARM      STAT: NO_ALARM
SVAL:
TPRO: 0
VAL: HEWLETT-PACKARD,E3631A,0,2.1-5.0-1.0

epics>
```

The SCPI `IDN*?` commands are run as part of iocInit since their records have `PINI=YES`.
If the stringin record has truncated the identification string you should be able to see the full string as a waveform record.
In another window run:

```console
norume> caget -S hpE3631ATestIDNwf
hpE3631ATestIDNwf HEWLETT-PACKARD,E3631A,0,2.1-5.0-1.0
norume>
```

## 4 - asynRecord support

The asynRecord provides a convenient mechanism for controlling the diagnostic messages produced by asyn drivers.
You can also use the MEDM asynOctet screen to manually send commands to
and view responses from a device.
This can be very handy when trying to figure out just how a new device actually works.

To use an asynRecord in your application:

Add the line:


```makefile
DB_INSTALLS += $(ASYN)/db/asynRecord.db
```

to the application Makefile.

Create the diagnostic record in the IOC by adding a line like:

```bash
dbLoadRecords("db/asynRecord.db","P=$(P)$(R),R=asyn,PORT=L0,ADDR=-1,OMAX=0,IMAX=0")
```

to the application startup script.
The `PORT` value must match the the value in the corresponding `drvAsynIPPortConfigure`
or `drvAsynSerialPortConfigure` command.
The `ADDR` value should be that of the instrument whose I/O you wish to monitor.

To run the asynRecord screen,
add `<asynTop>/opi/medm` to your `EPICS_DISPLAY_PATH` environment variable
and start medm with `P`, `R`, `PORT`
and `ADDR` values matching those given in the dbLoadRecords command:

```bash
medm -x -macro "P=hpE3631ATest,R=asyn,PORT=L0,ADDR=-1" asynRecord.adl
```

If you have edm installed
then you need to use the `asynRecord.edl` file instead.
Ensure that `<asynTop>/opi/medm` is in the `EDMDATAFILES` environment variable.
Start edm with the arguments appropriate for your IOC:

```bash
edm -x -m "P=hpE3631ATest, R=asyn,PORT=L1_TCP,ADDR=-1" asynRecord.edl
```

## 5 - Add records and protocol file entries

The descriptions in this section apply directly to the HP E3631A only but may provide useful hints for adding commands for your particular device.

### 5.1 - Add a record to set the device to local or remote control

First add an entry to `HPE3631ASup/devHPE3631A.proto`:

```
setLocRem {
    out "SYST:%{LOC|REM}";
}
```

This uses the StreamDevice ENUM converter to send one of two strings to the device based on the PV value. 
The corresponding addition to `HPE3631ASup/devHPE3631A.db` is:

```
record(bo, "$(P)$(R)SetRemote")
{
    field(DESC, "Set into local/remote mode")
    field(DTYP, "stream")
    field(OUT,  "@devHPE3631A.proto setLocRem $(PORT) $(A)")
    field(ZNAM, "Local")
    field(ONAM, "Remote")
    field(PINI, "YES")
    field(VAL,  "1")
}
```

The `PINI` and `VAL` fields ensure that the device is set into remote control mode during IOC startup.

Once you've made the above changes run `make` to install the modified files.
If you've created the test application you can try out the new command
and see if it works.

### - 5.2 Add records to set and check the output on/off status

These are simple since they use StreamDevice protocol entries that already exist.
The `setD` and `getD` entries set
and get the value of a single integer parameter whose command is passed as the argument to the protocol entry:

```
record(bo, "$(P)$(R)SetOnOff")
{
    field(DESC, "Turn output off/on")
    field(DTYP, "stream")
    field(OUT,  "@devHPE3631A.proto setD(OUTP) $(PORT) $(A)")
    field(ZNAM, "Off")
    field(ONAM, "On")
}
record(bi, "$(P)$(R)GetOnOff")
{
    field(DESC, "Is output on?")
    field(DTYP, "stream")
    field(INP,  "@devHPE3631A.proto getD(OUTP) $(PORT) $(A)")
    field(ZNAM, "Off")
    field(ONAM, "On")
}
```

### 5.3 - Add records to set and check the output voltage and current setpoints

Add two entries to `HPE3631ASup/devHPE3631A.proto`:

```
setVI {
    out "INST \$1;\$2 %g";
}
getVI {
    out "INST \$1;\$2?";
    in "%g";
    ExtraInput = Ignore;
}
```

These entries set and get the floating point value for the channel passed as the first argument
and for the parameter passed as the second argument.

The corresponding additions to `HPE3631ASup/devHPE3631A.db` are:

```
record(ao, "$(P)$(R)SetP6Volts")
{
    field(DESC, "Set +6V output volage")
    field(DTYP, "stream")
    field(OUT,  "@devHPE3631A.proto setVI(P6V,VOLT) $(PORT) $(A)")
    field(EGU,  "Volts")
    field(LOPR, "0")
    field(HOPR, "6.18")
    field(EGUL, "0")
    field(EGUF, "6.18")
    field(DRVL, "0")
    field(DRVH, "6.18")
}
record(ao, "$(P)$(R)SetP6Amps")
{
    field(DESC, "Set +6V output current")
    field(DTYP, "stream")
    field(OUT,  "@devHPE3631A.proto setVI(P6V,CURR) $(PORT) $(A)")
    field(EGU,  "Amps")
    field(LOPR, "0")
    field(HOPR, "5.15")
    field(EGUL, "0")
    field(EGUF, "5.15")
    field(DRVL, "0")
    field(DRVH, "5.15")
}
record(ao, "$(P)$(R)SetP25Volts")
{
    field(DESC, "Set +25V output volage")
    field(DTYP, "stream")
    field(OUT,  "@devHPE3631A.proto setVI(P25V,VOLT) $(PORT) $(A)")
    field(EGU,  "Volts")
    field(LOPR, "0")
    field(HOPR, "25.75")
    field(EGUL, "0")
    field(EGUF, "25.75")
    field(DRVL, "0")
    field(DRVH, "25.75")
}
record(ao, "$(P)$(R)SetP25Amps")
{
    field(DESC, "Set +25V output current")
    field(DTYP, "stream")
    field(OUT,  "@devHPE3631A.proto setVI(P25V,CURR) $(PORT) $(A)")
    field(EGU,  "Amps")
    field(LOPR, "0")
    field(HOPR, "1.03")
    field(EGUL, "0")
    field(EGUF, "1.03")
    field(DRVL, "0")
    field(DRVH, "1.03")
}
record(ao, "$(P)$(R)SetN25Volts")
{
    field(DESC, "Set -25V output volage")
    field(DTYP, "stream")
    field(OUT,  "@devHPE3631A.proto setVI(N25V,VOLT) $(PORT) $(A)")
    field(EGU,  "Volts")
    field(HOPR, "0")
    field(LOPR, "-25.75")
    field(EGUF, "0")
    field(EGUL, "-25.75")
    field(DRVH, "0")
    field(DRVL, "-25.75")
}
record(ao, "$(P)$(R)SetN25Amps")
{
    field(DESC, "Set -25V output current")
    field(DTYP, "stream")
    field(OUT,  "@devHPE3631A.proto setVI(N25V,CURR) $(PORT) $(A)")
    field(EGU,  "Amps")
    field(LOPR, "0")
    field(HOPR, "1.03")
    field(EGUL, "0")
    field(EGUF, "1.03")
    field(DRVL, "0")
    field(DRVH, "1.03")
}
record(ao, "$(P)$(R)GetP6Volts")
{
    field(DESC, "Get +6V output volage")
    field(DTYP, "stream")
    field(OUT,  "@devHPE3631A.proto getVI(P6V,VOLT) $(PORT) $(A)")
    field(EGU,  "Volts")
    field(LOPR, "0")
    field(HOPR, "6.18")
    field(EGUL, "0")
    field(EGUF, "6.18")
}
record(ao, "$(P)$(R)GetP6Amps")
{
    field(DESC, "Get +6V output current")
    field(DTYP, "stream")
    field(OUT,  "@devHPE3631A.proto getVI(P6V,CURR) $(PORT) $(A)")
    field(EGU,  "Amps")
    field(LOPR, "0")
    field(HOPR, "5.15")
    field(EGUL, "0")
    field(EGUF, "5.15")
}
record(ao, "$(P)$(R)GetP25Volts")
{
    field(DESC, "Get +25V output volage")
    field(DTYP, "stream")
    field(OUT,  "@devHPE3631A.proto getVI(P25V,VOLT) $(PORT) $(A)")
    field(EGU,  "Volts")
    field(LOPR, "0")
    field(HOPR, "25.75")
    field(EGUL, "0")
    field(EGUF, "25.75")
}
record(ao, "$(P)$(R)GetP25Amps")
{
    field(DESC, "Get +25V output current")
    field(DTYP, "stream")
    field(OUT,  "@devHPE3631A.proto getVI(P25V,CURR) $(PORT) $(A)")
    field(EGU,  "Amps")
    field(LOPR, "0")
    field(HOPR, "1.03")
    field(EGUL, "0")
    field(EGUF, "1.03")
}
record(ao, "$(P)$(R)GetN25Volts")
{
    field(DESC, "Get -25V output volage")
    field(DTYP, "stream")
    field(OUT,  "@devHPE3631A.proto getVI(N25V,VOLT) $(PORT) $(A)")
    field(EGU,  "Volts")
    field(LOPR, "-25.75")
    field(HOPR, "0")
    field(EGUL, "-25.75")
    field(EGUF, "0")
}
record(ao, "$(P)$(R)GetN25Amps")
{
    field(DESC, "Get -25V output current")
    field(DTYP, "stream")
    field(OUT,  "@devHPE3631A.proto getVI(N25V,CURR) $(PORT) $(A)")
    field(EGU,  "Amps")
    field(LOPR, "0")
    field(HOPR, "1.03")
    field(EGUL, "0")
    field(EGUF, "1.03")
}
```

Depending on your application you might set up `FLNK`
and/or `SCAN` fields to keep the setpoint readbacks up to date.
