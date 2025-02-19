## REYAX RYS352A GNSS Antenna module

https://reyax.com/products/RYS352A

Datasheet: https://reyax.com//upload/products_download/download_file/RYS352A.pdf

PAIR Command Guide: https://reyax.com//upload/products_download/download_file/RYS352x_PAIR_Command_Guide.pdf

## Defaults

Message from first boot:

```$PAIR021,REYAX_RYS3520_V2.6.0_20230323,S,N,7f013f8,2212301343,,,,,52596e7d,2212301341,a0aad3c,2212301342,,,-15.48,-15.48,-14.02,-15.48,0,0,##,0,1*48```

Default values of note:

```$PAIR863,1135*1D```

## How to configure this module

I was convinced that I had fried multiple modules (Did I use 5V logic on this by accident?). But I think that the GNSS chip is just working too hard and we have to race to get the chip to accept each command. If we put the chip into the "Stop GNSS" mode, then we're able to get responses for each command reliably. First we need to run the following command to disable sleep on GNSS when it is in the "Stop GNSS" mode:

```$PAIR382,1*2E```

You will have to spam this command fairly rappidly until you see the chip respond with ```$PAIR001,382,0*32```. Don't run the next command until you've seen this reply or else you'll put the chip to sleep and then you'll have to reboot the chip.

Next we run the following command to power off the GNSS features of the chip so we can configure it:

```$PAIR003*39```

This command needs to be spammed as well to get the chip to proccess it. At some point you'll see ```$PAIR001,003,0*38``` and the delugue of NMEA output will stop and you will only see output for your commands. At this point you can run any commands you wish, don't forget to save the config with ```$PAIR513*3D```, and then you can either power cycle the module or you can run ```$PAIR002*38``` to put the module back into the same mode it was before you ran PAIR003. 


## How to enable RTCM

The datasheet says that this module supports RTCM output, and the config guide has multiple commands to toggle various RTCM outputs, but I could NOT get it to actually output any RTCM. I found the solution by searching across multiple Airoha based module vendors, and got the biggest hints on the Quectel forums with a [few](https://forums.quectel.com/t/firmware-request-lc29hea-adjustable-update-rate-for-ardupilot-utilization/33839/11) [different](https://forums.quectel.com/t/beginner-question-on-rtcm-on-lc29/36113/19) [posts](https://forums.quectel.com/t/qgnss-v1-10-does-not-seem-to-apply-rtk-data-to-lc29hea-fixes/34595/2) by [Marco](https://forums.quectel.com/u/bamarcant/summary) mentioning the ```$PAIR862``` and ```$PAIR863``` commands. The only document I could find that covers these commands is the [Simcom](https://www.simcom.com/) ``SIM68D Series NMEA Message User Guide``. This isn't available publicly from their website, but a vendor was nice enough to [provide a copy](https://web.archive.org/web/2/https://mt.morepower.ru/sites/default/files/documents/sim68d_series_nmea_message_user_guide_v1.01.pdf).

So in addition to the ```$PAIR382,1*2E``` & ```$PAIR003*39``` commands I explained above, the following are the commands I use to configure a REYAX RYS352A to output RTCM messages:

### Enable outputting RTCM messages to UART0

```$PAIR862,0,0,127*2E```

#### To check configured value:

```$PAIR863,0,0*37```

### Set Baudrate to 460800

```$PAIR864,0,0,460800*16```

### Set navigation mode.
* '0' Normal mode: For general purpose
* '1' [Default Value] Fitness mode: For running and walking activities so that the low-speed (< 5 m/s) movement will have more of an effect on the position calculation.
* '2' Reserved
* '3' Balloon mode: For high-altitude balloon purpose that the vertical movement will have more effect on the position calculation.
* '4' Stationary mode: For stationary applications where a zero dynamic assumed.
* '5' Drone mode: Used for drone applications with equivalent dynamics range and vertical acceleration on different flight phase. (Ex. hovering, cruising, etc.) (Note: The NMEA decimal precision will change to lat/lon in 7 digits, alt in 3 digit automatically)
* '6' Reserved
* '7' Swimming mode: For swimming purpose so that it smooths the trajectory and improves the accuracy of distance calculation.

```$PAIR080,4*2A```

#### To check configured value:

```$PAIR081*33```

### Sets RTCM output mode.
* -1 = Disable outputting RTCM
* 0 = Enable output RTCM3 with message type MSM4
* 1 = Enable output RTCM3 with message type MSM7
 
```$PAIR432,1*22```

#### To check configured value:

```$PAIR433*3E```

### Enables/disables outputting stationary antenna reference point in RTCM format.
RTCM3 Message 1005

```$PAIR434,1*24```

#### To check configured value:

```$PAIR435*38```

### Enables/disables outputting satellite ephemeris in RTCM format.
RTCM3 Messages 1019, 1020, 1042, 1044, 1046

```$PAIR436,1*26```

#### To check configured value:

```$PAIR437*3A```

### Saves the current configurations from RTC RAM to NVM.

```$PAIR513*3D```


At this point you need to power cycle the module, and reconfigure your software to use 460800 as the baud rate.
