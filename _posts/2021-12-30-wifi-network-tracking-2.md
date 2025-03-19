---
layout: post
title: WiFi Network Tracking - Imitating Google's Geolocation Network (Part 2)
description: Having recorded WiFi network geolocation information, I'll now be writing software to filter and manipulate that data. This will allow us to locate WiFi networks using just their SSID or BSSID.
readtime: 10 minute
toc: true
tags: programming, cybersecurity
---

<script type="text/javascript" src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML"></script>

In part one of this mini-project, we wrote WiFi network BSSIDs, SSIDs and dBm signals to a file along with the latitude and longitude of where we recorded that information - all in a json format. Since then I have gone out and collected location data on many networks near me using the python3 program we wrote. Now it's time to put that data to use.

## dBm Signal
One of the most critical pieces of information we recorded was the dBm signal of the WiFi networks, measured from the point of recording. Due to the fact that dBm represents signal strength, we can use it to derive distance between the point of recording and the WiFi network. Some great diagrams showing dBm over distance can be found [here](https://semfionetworks.com/blog/free-space-path-loss-diagrams/).

In part one of this mini-project, I referenced [this python2 script](https://gist.github.com/cryptolok/516471ce35a9851197b204853c6de080) that converts dBm into a distance in metres. Researching more, I found disagreement on [this stackoverflow thread](https://stackoverflow.com/questions/11217674/how-to-calculate-distance-from-wifi-router-using-signal-strength) in attempt to determine the correct equation. Some suggest that an [absolute value](https://en.wikipedia.org/wiki/Absolute_value) of dBm should be used, this is correct due to the dBm range of received signal from a wireless network being -10 to -100 (always negative). If an absolute value wasn't used, the distance returned by the equation would always be negative.

There are tweaks that could be performed when implementing the equation, such as using the [exact frequency of the channel](https://en.wikipedia.org/wiki/List_of_WLAN_channels#2.4_GHz_(802.11b/g/n/ax)) that the WiFi network was discovered on. But I'm not going to be implementing that here as I don't expect the effect to be too drastic.

Here are the mathematical equations I'll be using to calculate the distance (in metres) from dBm and frequency for now:

$${exp} = \frac{27.55 - 20 \times {\log_{10}(frequency)} + {\mid dBm \mid}}{20}$$

$${distance_m} = 10^{exp}$$

However, there is a pitfall to this equation, and to understand it I need to explain a bit about dB and dBm. dB (or decibel) is the standard unit to measure intensity of sound or power level of an electrical signal **by comparing it** with a given level on a scale. This means that **dB is a ratio** and **not a concrete measurement**. dBm stands for decibels per milliwatt and **is a concrete measurement** due to the fact it compares the dB to one milliwatt of power. See: [dB vs dBm in the context of phone signal](https://www.signalboosters.com/blog/what-is-dbm/).

In the equation, instead of dBm we should be using dB. But we have no way of getting the decibel value, because dB and dBm are inconvertible types. This introduces an inaccuracy in our current equation which we will address later.

## Python Scripts
### Searching The Data
I wrote a python3 script to help me search the networks I had recorded. This simply printed out all the entries where the BSSID or SSID contained my search term:

```python
import json

search_term = input('Input BSSID or SSID to search for: ')
found_entries = {'bssid': [], 'ssid': []}

with open('scan_data', 'r') as f:
	content = f.read()
lines = content.split('\n')
for line in lines:
	if len(line) < 4:
		continue
	entry = json.loads(line)
	if search_term in entry['bssid']:
		found_entries['bssid'].append(entry)
	elif search_term in entry['ssid']:
		found_entries['ssid'].append(entry)

print('')
print('Entries found by SSID:\n')
for i in found_entries['ssid']:
	print(i)
print('')
print('Entries found by BSSID:\n')
for i in found_entries['bssid']:
	print(i)
print('')
```

### dBm To Metre Calculation
Here's the python implementation of the dBm to distance (metres) equation we discussed previously:

```python
from math import log10

dBm = int(input('Input dBm: '))
MHz = 2417
FSPL = 27.55
dist = 10 ** ((FSPL - (20 * log10(MHz)) + abs(dBm)) / 20)
dist = round(dist, 2)
print(dist)
```

### Searching & Calculating
I combined the two above scripts to produce one which could search the data and calculate the distance for each found entry, then print out the results:

```python
import json
from math import log10

class EntryManager:
    def __init__(self):
        self.entries = []

    def load_entries(self):
        with open('scan_data', 'r') as f:
            content = f.read()
        lines = content.split('\n')
        for line in lines:
            if len(line) < 4:
                continue
            entry = json.loads(line)
            self.entries.append(entry)

    def search_entries(self, search_term):
        found_entries = []
        for entry in self.entries:
            if entry['bssid'] == search_term or entry['ssid'] == search_term:
                found_entries.append(entry)
        return found_entries
    
    def select_network_bssid(entries):
        unique_entries = {}
        for entry in entries:
            if entry['bssid'] in unique_entries:
                unique_entries[entry['bssid']] = unique_entries[entry['bssid']] + 1
            else:
                unique_entries[entry['bssid']] = 1
        if len(unique_entries.keys()) == 0:
            return ''
        elif len(unique_entries.keys()) == 1:
            return list(unique_entries.keys())[0]
        print('Matching Networks:')
        for bssid in unique_entries.keys():
            print('\t' + bssid + ' (' + str(unique_entries[bssid]) + ' entries)')
        return input('Enter the BSSID: ')
    
    def filter_by_bssid(entries, bssid):
        filtered = []
        for entry in entries:
            if entry['bssid'] == bssid:
                filtered.append(entry)
        return filtered

    def calc_dist_entries(entries):
        for entry in entries:
            dBm = entry['dBm']
            MHz = 2417
            dist = 10 ** ((27.55 - (20 * log10(MHz)) + abs(dBm)) / 20)
            dist = round(dist, 2)
            entry['dist_m'] = dist
        return entries

entry_manager = EntryManager()
entry_manager.load_entries()
search_term = input('Input BSSID or SSID to search for: ')
entries = entry_manager.search_entries(search_term)
if len(entries) == 0:
    print('No found entries')
    exit()
bssid = EntryManager.select_network_bssid(entries)
entries = EntryManager.filter_by_bssid(entries, bssid)
entries = EntryManager.calc_dist_entries(entries)
print(entries)
```

This was great progress. Now all I needed to do was visualise this data on a map.

### Map Visualisation
I found [this great guide](https://www.tutorialspoint.com/plotting-google-map-using-gmplot-package-in-python) detailing how to utilise the `gmplot` python package to draw geographical locations using google maps. I'd recommend giving it a read as I'm not going to be explaining most of my code for it.

I implemented this into my current solution as well as adding a few other changes to produce the following:

```python
import json
from math import log10
from gmplot import gmplot

class EntryManager:
    def __init__(self):
        self.entries = []

    def load_entries(self):
        with open('scan_data', 'r') as f:
            content = f.read()
        lines = content.split('\n')
        for line in lines:
            if len(line) < 4:
                continue
            entry = json.loads(line)
            self.entries.append(entry)

    def search_entries(self, search_term):
        found_entries = []
        for entry in self.entries:
            if entry['bssid'] == search_term or entry['ssid'] == search_term:
                found_entries.append(entry)
        return found_entries
    
    def select_network_bssids(entries):
        unique_entries = {}
        for entry in entries:
            if entry['bssid'] in unique_entries:
                unique_entries[entry['bssid']] = unique_entries[entry['bssid']] + 1
            else:
                unique_entries[entry['bssid']] = 1
        if len(unique_entries.keys()) == 0:
            return ''
        elif len(unique_entries.keys()) == 1:
            return list(unique_entries.keys())[0]
        print('Matching Networks:')
        for bssid in unique_entries.keys():
            print('\t' + bssid + ' (' + str(unique_entries[bssid]) + ' entries)')
        bssid = input('Enter the BSSID: ')
        if bssid == 'all':
            return list(unique_entries.keys())
        else:
            return [bssid]
    
    def filter_by_bssids(entries, bssids):
        filtered = []
        for entry in entries:
            if entry['bssid'] in bssids:
                filtered.append(entry)
        return filtered

    def calc_dist_entries(entries):
        for entry in entries:
            dBm = entry['dBm']
            MHz = 2417
            dist = 10 ** ((27.55 - (20 * log10(MHz)) + abs(dBm)) / 20)
            dist = round(dist, 2)
            entry['dist_m'] = dist
        return entries

class Circle:
    def __init__(self, latitude, longitude, radius):
        self.latitude = latitude
        self.longitude = longitude
        self.radius = radius

class MapPlotter:
    def __init__(self, entries):
        self.entries = entries

    def get_circles(self):
        circles = []
        for entry in self.entries:
            c = Circle(entry['latitude'], entry['longitude'], entry['dist_m'])
            circles.append(c)
        to_remove = []
        for c1 in circles:
            for c2 in circles:
                if c1 == c2:
                    continue
                if MapPlotter.is_circle_in_circle(c1, c2):
                    to_remove.append(c2)
        for c in to_remove:
            if c in circles:
                circles.remove(c)
        return circles

    def draw(self, out_file):
        circles = self.get_circles()
        gmap = gmplot.GoogleMapPlotter(circles[0].latitude, circles[0].longitude, 16)
        #gmap.apikey = 'REDACTED'
        for c in circles:
            gmap.circle(c.latitude, c.longitude, c.radius, face_alpha=0.2, face_color='#00FFFF')
        gmap.draw(out_file)
    
    def is_circle_in_circle(c1, c2):
        distSq = (((c1.latitude - c2.latitude)*(c1.latitude - c2.latitude))+((c1.longitude - c2.longitude)* (c1.longitude - c2.longitude)))**(.5)
        if (distSq + c2.radius <= c1.radius):
            return False
        else:
            return True

entry_manager = EntryManager()
entry_manager.load_entries()
search_term = input('Input BSSID or SSID to search for: ')
entries = entry_manager.search_entries(search_term)
if len(entries) == 0:
    print('No found entries')
    exit()
bssids = EntryManager.select_network_bssids(entries)
entries = EntryManager.filter_by_bssids(entries, bssids)
entries = EntryManager.calc_dist_entries(entries)
map_plotter = MapPlotter(entries)
map_plotter.draw('map.html')
```

The circles on the drawn map represent the possible area of the selected WiFi network. This is determined by drawing a circle with the centre being the latitude and longitude of the recording location, and the radius of the circle being the distance away from the WiFi network in metres (calculated using dBm).

Also, using the math proposed by [this article](https://www.geeksforgeeks.org/check-if-a-circle-lies-inside-another-circle-or-not/) allowed me to check if a circle was fully inside of another circle. This was implemented to ensure we see the most accurate possible readings and reduce the number of massive inaccurate circles. My implementation isn't perfect thought, there are cases where circles are hidden, despite the fact that they further decrease the possible network area. However, creating a perfect solution would be very difficult, time-consuming and overkill for what we are trying to achieve here.

A change I made to the solution in general, was giving the ability to use multiple WiFi networks when drawing the map - this was done to support mesh networks and is handled by the `select_network_bssids` method. 

## Initial Demonstration
By driving down the yellow road in the image below, I captured 5 entries for a network which, when entered into the program, produced the below map (all text has been removed for privacy and security). The blue circle represents the area where the WiFi network could be located. I have also denoted the actual location of the WiFi network with a red cross for comparison.

<img src="/assets/img/wifi-network-tracking-map-1.png"/>

## Improved dBm To Metre Calculation
As previously mentioned, we have an inaccuracy in our dBm to metre calculation due to the use of dBm instead of dB. To solve this we can use a more specific equation as suggested by [this stackoverflow answer](https://stackoverflow.com/a/27238116/14844306). Here we specify every single variable that makes up the Free-Space Path Loss (FSPL) and use an altered equation. Here are the equations we're going to use (variables explained in [the stackoverflow answer](https://stackoverflow.com/a/27238116/14844306)):

$$K = -27.55$$

$$FSPL = Ptx - CLtx + AGtx + AGrx - CLrx - Prx - FM$$

$$exp = \frac{FSPL - K - 20 \times \log_{10}(frequency)}{20}$$

$${distance_m} = 10^{exp}$$

I implemented the equations by amending the `calc_dist_entries` method in our previous solution:

```python
def calc_dist_entries(entries):
	for entry in entries:
		Prx = entry['dBm']
		frequency = 2417
		K = -27.55
		Ptx = 16
		AGtx = 2
		AGrx = 0
		CLtx = 0
		CLrx = 0
		FM = 22

		FSPL = Ptx - CLtx + AGtx + AGrx - CLrx - Prx - FM
		d = 10 ** ((FSPL - K - 20 * log10(frequency)) / 20)
		d = round(d, 2)
		entry['dist_m'] = d
	return entries
```

and remembering to add the following import:

```python
from math import log10
```

## Final Demonstration
Using the same network as tested in the initial demonstration, but with our improved distance calculator, the below map was produced (all text has been removed for privacy and security). The blue circle represents the area where the WiFi network could be located. I have also denoted the actual location of the WiFi network with a red cross.

<img src="/assets/img/wifi-network-tracking-map-2.png" />

Here we see a much smaller and more accurate possible area for the WiFi network.

## Conclusion
#### Legality
What we have done here is completely legal, most phones and IOT devices produced by companies such as Google have this functionality built in. After all, we have only checked our surrounding WiFi networks and recorded their location.

#### Ethicality
For this project, I believe the ethicality of the system strongly depends upon how it is used. For example, it could become part of an advanced guidance system for civilians - I'd perceive this as an ethical application. However, it could also become part of a piece of malware which uses location networks and surrounding WiFi networks to pinpoint target location (similar to [HoneyBadger](https://github.com/lanmaster53/honeybadger/)) - I'd perceive this as an unethical application (assuming the malware is used with malicious intent).

This project is merely a tool which can be utilised in many different ways. Companies such as [SkyHook](https://www.skyhook.com/) implement similar solutions to what we have built here and provide advanced location services for businesses - who each may have very different use cases.

#### Closing Thoughts
Overall, our solution isn't perfect, and obviously isn't as sophisticated as Google Maps' location network. But that's alright, this is just a proof of concept to demonstrate the capabilities of this kind of technology - and with more data, it'd get more precise.

Thanks for reading, feel free to contact me with any queries about this mini-project: `contact@benanderson.xyz`
