---
layout: post
title: WiFi Network Tracking - Imitating Google's Geolocation Network (Part 1)
description: Breaking down Google's international geolocation network and building a proof of concept from the ground up using intermediate cyber security and programming skills - explaining design decisions made along the way.
readtime: 9 minute
tags: programming, cybersecurity
---

It's well known that Google builds WiFi router discovery into many of their products - most notably android. This gives them the ability to locate almost any WiFi router in the world.<!--excerpt--> Using IP address for location tracking is very much inferior to recording the locations of WiFi routers and using that to determine the location of an internet-connected device. Google's geolocation api using WiFi router BSSIDs can give location to an accuracy of around 20 metres (on average).

I got the idea to do this project when watching [John Hammond's video](https://youtu.be/7LXfBSuaFFE?t=2169) with John Strand who talked a bit about the cyber security tool [HoneyBadger](https://github.com/lanmaster53/honeybadger/).

## The Idea
My idea for this project is to have a go at logging WiFi router locations using WiFi signal strength and a starting GPS location, then using the data to track location (which will include some data manipulation and triangulation). I merely hope to see the plausability of a normal person deploying and experimenting with such technology and will not be building any sort of large scalable infrastructure like Google.

## Method of Approach
I started off by looking at using a Raspberry Pi for this project as it's very portable - this would help when recording and saving WiFi router BSSIDs (unique address that identifies an access point). I also found [this beginner friendly guide](https://www.raspberrypi.org/app/uploads/2017/10/OYCSU_GPS.v2-1.pdf) on raspberrypi's website which describes how to record GPS coordinates of the raspberry pi in real time using a USB GPS dongle and the `pigps` python library.

I have previously looked at WiFi lots when researching cyber security so I was already familiar with a tool called `airodump-ng` which allows you to sniff packets - including the BSSID and signal strength of access points. And at first, this was the tool I planned to use for this project. However, when researching I came across [this great resource](https://www.thepythoncode.com/article/building-wifi-scanner-in-python-scapy) which details using the python library `scapy` to list wireless access points.

The WiFi scanning article also shows how to get the [dBm signal](https://en.wikipedia.org/wiki/DBm) of the access point. Having previously researched dBm in the context of bluetooth, I was aware of [this script](https://gist.github.com/cryptolok/516471ce35a9851197b204853c6de080) written in python2 which converts a dBm signal into distance (in metres) for a given frequency and so would use that too.

Using the above information, I planned to take the raspberry pi out on a walk and do the following:
1. Record an access point's BSSID (using `scapy`)
2. Record signal strength of an access point (using `scapy`)
3. Record GPS coordinates of the raspberry pi  (using `pigps`)

Combining this information would hopefully give me enough data to triangulate the position of any WiFi access point I had recorded - assuming I recorded enough data.

## Prototype Recording Program
For the prototype, I only wanted to write the recorded data to a local file on the raspberry pi for sake of simplicity and testing, but I recognised that using a device such as the [Hologram Nova](https://www.hologram.io/products/nova) in the future to easily get an internet connection on the raspberry pi from anywhere and stream the data would be a nice development.

After reading through [this article](https://www.thepythoncode.com/article/building-wifi-scanner-in-python-scapy) on WiFi scanning with scapy, I adapted their code to produce the following prototype test for recording access point BSSIDs, SSIDs and dBms:

```python
from scapy.all import *
from threading import Thread
import time
import os

def callback(packet):
	if packet.haslayer(Dot11Beacon):
		bssid = packet[Dot11].addr2
		ssid = packet[Dot11Elt].info.decode()
		try:
			dbm_signal = packet.dBm_AntSignal
		except:
			dbm_signal = "N/A"
		print({'bssid': bssid, 'ssid': ssid, 'dBm': dbm_signal})

def change_channel():
	ch = 1
	while True:
		# switch channel from 1 to 14 each 0.5s
		os.system(f"iwconfig {interface} channel {ch}")
		ch = ch % 14 + 1
		time.sleep(0.5)  

if __name__ == "__main__":
	interface = "wlan0"
	channel_changer = Thread(target=change_channel)
	channel_changer.daemon = True
	channel_changer.start()
	sniff(prn=callback, iface=interface)
```

This yielded the following output of data (access point BSSIDs and SSIDs have been changed for privacy and security):

```json
...
{"bssid": "48:e7:6e:5d:1d:87", "ssid": "SKY18466", "dBm": -81}
{"bssid": "a6:6c:c5:b0:07:09", "ssid": "SKY56981", "dBm": -88}
{"bssid": "94:17:65:00:1d:4d", "ssid": "Virgin Media", "dBm": -42}
{"bssid": "e7:1b:a9:4d:ee:b0", "ssid": "VM2450213-2G", "dBm": -50}
{"bssid": "39:e4:90:38:d2:76", "ssid": "Virgin Media", "dBm": -82}
...
```

All results were as expected. If you did not get similar results from trying this then please note you need a wireless network adapter that supports monitor mode, and is in monitor mode (see: [enabling monitor mode](https://linuxhint.com/monitor_mode_kali_linux_2020/)). I am personally using the [Alfa Network AWUS036NHA USB WiFi Adapter](https://www.amazon.co.uk/gp/product/B004Y6MIXS/).

Now time for logging GPS location. I came across [this friendly guide](https://www.raspberrypi.org/app/uploads/2017/10/OYCSU_GPS.v2-1.pdf) on the Raspberry Pi Foundation's website, detailing how to use a GPS dongle to geolocate yourself. I purchased [this GPS dongle](https://www.amazon.co.uk/ALAMSCN-G-mouse-Glonass-Raspberry-Compatible-White/dp/B09JJYD978/) and used it with the script in said article. However, for the first few seconds I ran the script I saw `0` as the latitude and longitude, and suspected it wasn't working. But in actual fact after around 10 seconds it began to function as expected, so that's something to look out for if you plan on utilising GPS geolocation with a GPS dongle.

I adapted the script in the article a little and tested the following:

```python
from pigps import GPS
import time

gps = GPS()

while True:
	if gps.lat == 0 or gps.lon == 0:
		print('Loading GPS...')
		time.sleep(1)
	else:
		print('{' + f'\"latitude\": {gps.lat}, \"longitude\": {gps.lon}' + '}')
		time.sleep(1)
```

The script first prints 'Loading GPS...' for a few seconds then starts outputting latitude and longitude data as expected.

## Building The Recording Program From The Prototypes
Next I decided to combine the WiFi scanner test script with the GPS test script to produce a working prototype:

```python
from pigps import GPS
from scapy.all import *
from threading import Thread
from sys import argv
import time
import os
import logging
import json

# script only supports linux due to command usage and pigps library

# dictionary to help reduce duplicate data (not 100% duplicate reducing yet)
last_locs = {}

def callback(packet):
	if not packet.haslayer(Dot11Beacon):
		return

	#get bssid, ssid and dBm of network
	bssid = packet[Dot11].addr2
	ssid = packet[Dot11Elt].info.decode()
	try:
		dbm_signal = packet.dBm_AntSignal
	except:
		logging.debug('dropping packet due to no dBm signal')
		return

	#round latitude and longitude to 5 decimal places
	#see https://lm.solar/heliostats/support/decimal-latitude-longitude-accuracy/
	latlon_dp_acc = 5
	lat = round(gps.lat * 10**latlon_dp_acc) / 10**latlon_dp_acc
	lon = round(gps.lon * 10**latlon_dp_acc) / 10**latlon_dp_acc
	if lat == 0 or lon == 0:
		logging.debug('dropping packet due to gps not loaded (lat or long == 0)')
		return

	#try reduce duplicate data if last logging location for network same as current
	if bssid in last_locs and last_locs[bssid] == (lat, lon):
		logging.debug(f'dropping packet due to gps location same as last entry for BSSID {bssid}')
		return

	#log network information
	last_locs[bssid] = (lat, lon)
	entry = {'bssid': bssid, 'ssid': ssid, 'dBm': dbm_signal, 'latitude': lat, 'longitude': lon}
	logging.debug(f'recording entry {entry}')
	with open('scan_data', 'a') as data_file:
		data_file.write(json.dumps(entry) + "\n")

def change_channel():
	ch = 1
	while True:
		# switch channel from 1 to 14 each 0.5s
		os.system(f"sudo iwconfig {interface} channel {ch}")
		logging.debug(f'switched wireless adapter {interface} to channel {ch}')
		ch = ch % 14 + 1
		time.sleep(0.5)

if __name__ == "__main__":
	#create logging file and config
	log_file = 'scan-log_' + time.strftime('%Y-%m-%d_%H-%M-%S') + '.log'
	logging.basicConfig(filename=log_file, format='[%(asctime)s] [%(levelname)s] %(message)s', level=logging.DEBUG)
	
	#take wireless interface in as command line argument
	if len(argv) > 1:
		interface = argv[1]
	else:
		logging.error('Please specify wireless interface')
		print('[!] Please specify wireless interface')
		exit()
	
	#put wireless interface into monitor mode
	logging.info('Putting wireless interface into monitor mode...')
	os.system(f"sudo ifconfig {interface} down")
	os.system(f"sudo iwconfig {interface} mode monitor")
	os.system(f"sudo ifconfig {interface} up")
	logging.info('Wireless interface put into monitor mode')
	
	#start GPS
	logging.info('Launching GPS...')
	gps = GPS() #if doesn't work try passing 'dev' parameter with path to gps dongle
	logging.info('GPS Launched')
	
	#start wireless adapter channel switcher	
	logging.info('Launching wireless adapter channel switcher...')	
	channel_changer = Thread(target=change_channel)	
	channel_changer.daemon = True	
	channel_changer.start()	
	logging.info('Wireless adapter channel switcher launched')

	#start sniffing packets to find networks	
	logging.info('Started sniffing packets')	
	sniff(prn=callback, iface=interface)
```

I made a few noteworthy modifications other than just combining the previous test scripts.

Firstly, I've added a logger using python's basic `logging` library. The code for that is fairly self-explanatory and there is lots of [documentation](https://docs.python.org/3/howto/logging.html) for it online if you would like to know more about it. Also, all scan results are now written to the file `scan_data` in a json format, instead of just being printed out.

Secondly, the wireless interface used by the WiFi scanner is now passed into the program as an argument, for example: `python3 wifi_scan_gps.py wlan0`.  Furthermore, the script now automatically enables monitor mode on the wireless network adapter if it can, instead of having to enable it manually before running the script.

Finally, I have rounded latitude and longitude values read from the GPS dongle to 5 decimal places, as this left a nice accuracy of around 3.6 feet (according to [this page on LightManufacturing's website](https://lm.solar/heliostats/support/decimal-latitude-longitude-accuracy/)). This allowed me to use a dictionary (which I called `last_locs`) to store the last GPS location we recorded from for each access point. This is because if we record the same access point from the same GPS location there is no need to log anything as it will be duplicate data, the dictionary helps store less of this.

I ran the prototype whilst stationary and got lots data, although due to how identifiable it is, there's really no point in making it all fake and posting it here.

So far I've been running all the tests on my laptop and attaching the equipment up to that. I'll be sticking with that for now but also ensuring all tools and hardware I use would be compatible with the raspberry pi. The next stage of this project will be to take the laptop out in the car (not driving myself) to gather more data, in preparation for doing some data manipulation and position triangulation.