FlexThing - FlexRadio Controller for DeskThing

https://www.youtube.com/watch?v=11c7C-oZdTI - Spotify Car Thing: Now My Favorite Ham Shack Tool


A DeskThing application that transforms your Spotify Car Thing into a dedicated control surface for FlexRadio 6000/8000 series SDR transceivers.
Features
VFO Control

Real-time frequency display with band indicator
Scroll wheel tuning with configurable step sizes (1 Hz, 10 Hz, 100 Hz, 1 kHz, 10 kHz)
Band up/down navigation across all HF bands (160m - 6m)
Mode selection (USB, LSB, CW, AM, FM, DIGU, DIGL, SAM, NFM, RTTY)

Live Metering

Real-time S-meter display via UDP streaming
Signal level in dBm with visual bar graph

DSP Controls

Noise Blanker (NB) toggle
Noise Reduction (NR) toggle
Dedicated DSP screen for quick access

TX Features

TX/RX status indicator
Power level adjustment (up/down)
ATU tune trigger
Antenna switching (ANT1, ANT2, XVTR, RX_A, RX_B)

Memory System

8 memory slots for frequency/mode storage
Quick recall from dedicated memory screen

POTA Integration

Live Parks on the Air spot feed
Displays activator callsign, frequency, park reference, and location
One-touch tune to spotted frequency
Auto-refresh every 60 seconds

Multi-Screen Interface

VFO Screen: Main frequency display and tuning
DSP Screen: NB/NR controls
Memory Screen: 8 memory slot buttons
TX Screen: Power and antenna controls
POTA Screen: Live spot browser

Requirements

DeskThing Server v0.11.0 or higher
Spotify Car Thing with DeskThing client
FlexRadio 6000 or 8000 series transceiver
Network connection between PC and radio

Installation

Download the latest flex-thing-vX.X.X.zip from Releases
Open DeskThing Server
Go to Downloads → Install from ZIP
Select the downloaded file
Configure radio IP (or leave blank for auto-discovery)

Configuration
SettingDescriptionDefaultRadio IPFlexRadio IP address (blank for auto-discovery)Auto
Technical Details

Protocol: FlexRadio SmartSDR TCP API (port 4992)
Meter Data: VITA-49 UDP streaming (port 54993)
Discovery: UDP broadcast on port 4992
API Version: Compatible with SmartSDR v1.4+

Physical Button Mapping
All functions can be mapped to the Car Thing's physical buttons:

Tune Up/Down
Band Up/Down
Cycle Step Size
Cycle Mode
Toggle TX
ATU Tune
Switch Antenna
Toggle NB/NR
Power Up/Down
Memory Recall (1-8)
Screen Navigation
POTA Scroll

Building from Source
bash# Clone repository
git clone https://github.com/yourusername/flex-thing.git
cd flex-thing

# Install dependencies
npm install

# Build
npm run build

# Output: dist/flex-thing-vX.X.X.zip
Version History

v1.8.6 - Handler architecture fix, eliminates all double-action bugs
v1.8.2 - S-meter UDP streaming with correct signed 16-bit scaling
v1.7.9 - Standard tuning steps, 3-digit Hz display
v1.5.0 - POTA integration, multi-screen UI
v1.0.0 - Initial release

License
MIT License
Acknowledgments

DeskThing by ItsRiprod
FlexRadio Systems for the SmartSDR API
Parks on the Air for the spots API
