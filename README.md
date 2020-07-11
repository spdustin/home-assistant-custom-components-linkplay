# LinkPlay devices integration v2

This component allows you to integrate control of audio devices based on LinkPlay chipset into your [Home-Assistant](http://www.home-assistant.io) smart home system. Originally developed by nicjo814, maintained by limych. This version partly rewritten by nagyrobi. Read more about LinkPlay at the bottom of this file.

Fully compatible with [Mini Media Player card for Lovelace UI](https://github.com/kalkih/mini-media-player) by kalkih, including speaker group management.

## Installation
* Install manually: copy all files in custom_components/linkplay to your <config directory>/custom_components/linkplay/ directory.
* Restart Home-Assistant.
* Add the configuration to your configuration.yaml.
* Restart Home-Assistant again.

### Warning !!!
This **will overwrite** the previous **LinkPlay Sound Devices Integration** component if you had it installed. Also the configuration settings are not backwards compatible so **you will have to adjust** them as documented below otherwise it may break your system. To avoid this, make a backup of your previous linkplay config and remove it from your Home Assistant instance. Also uninstall/delete the previous linkplay component and restart Home Assistant.

[Support forum](https://community.home-assistant.io/t/linkplay-integration/33878/133)

### Example configuration entry

1. To add LinkPlay device to your installation, add the following to your `configuration.yaml` file:

    ```yaml
    # Example configuration.yaml entry
    media_player:
        - platform: linkplay
          host: 192.168.1.11
          name: Sound Room1
          device_name: Room1
          icecast_metadata: 'StationNameSongTitle'
          unavailable_message: '<l'appareil n'est pas disponible>'
          multiroom_wifidirect: False
          sources: 
            {
              'wifi': 'WiFi', 
              'optical': 'TV sound', 
              'line-in': 'Radio tuner', 
              'bluetooth': 'Bluetooth',
              'udisk': 'USB HDD'
              'http://94.199.183.186:8000/jazzy-soul.mp3': 'Jazzy Soul',
            }

        - platform: linkplay
          host: 192.168.1.12
          name: Sound Room2
          device_name: Room2
          icecast_metadata: 'Off'  # valid values: 'Off', 'StationName', 'StationNameSongTitle'
          unavailable_message: '<Das Gerät ist nicht verfügbar>'
          sources: 
            {
              'wifi': 'WiFi'
            }
    ```

### Configuration Variables

**host:**\
  *(string)* *(Required)* The host name or IP address of the Linkplay unit.

**name:**\
  *(string)* *(Required)* Name to use in the frontend. Nothe that Home Assistant will generate the `entity_id` based on this.

**device_name:**\
  *(string)* *(Optional)* The name of the device, as it setted up in the official application. Recent firwmares will publish this, if so, it will be overwitten from there.

**icecast_metadata:**\
  *(string)* *(Optional)* When playing icecast webradio streams, how to handle metadata. Valid values here are `'Off'`, `'StationName'`, `'StationNameSongTitle'`, defaulting to `'StationName'` when not set. With `'Off'`, Home Assistant will not try do request any metadata from the IceCast server. With `'StationName'`, Home Assistant will request from the headers only once when starting the playback the stream name, and display it in the `media_title` property of the player. With `'StationNameSongTitle'` Home Assistant will request the stream server periodically for icy-metadata, and read out `StreamTitle`, trying to figure out correct values for `media_title` and `media_artist`, in order to gather cover art information from LastFM service (see below). The stream name (usually the name of the radio station) will be placed in the `friendly_name` property of the player while playing. Note that this depends on how the icecast radio station servers and encoders are configured, if they don't provide proper metadata, it's better to turn it off or just use StationName to save server load.

**unavailable_message:**\
  *(string)* *(Optional)* Override the default `<unable to connect to device>` text message shown if the device has connection issues with the network.

**multiroom_wifidirect:**\
  *(boolean)* *(Optional)* Set to `True` to override the default router mode of multiroom connection, and force the old wifi-direct method. Units with firmware version lower than v4.2.8020 will be autodetected and they connect to multirooms only in wifi-direct mode. This option is needed only for units with newer versions if the user has a mix of players running old and new firmware.

**lastfm_api_key:**\
  *(string)* *(Optional)* API key to LastFM service to get album covers. Register for one.

**sources:**\
  *(list)* *(Optional)* A list with available sources on the device. If not specified, the integration will assume all sources are present on it:
```yaml
'wifi': 'WiFi', 
'line-in': 'Line-in', 
'line-in2': 'Line-in2', 
'bluetooth': 'Bluetooth', 
'optical': 'Optical', 
'rca': 'RCA', 
'co-axial': 'S-PDIF', 
'tfcard': 'SD',
'hdmi': 'HDMI',
'xlr': 'XLR', 
'fm': 'FM', 
'cd': 'CD', 
'udisk': 'USB'
```
The sources can be renamed to your preference (change only the part after **:** ). You can also specify http-based (Icecast / Shoutcast) internet radio streams as input sources:
```yaml
'http://1.2.3.4:8000/your_radio': 'Your Radio',
'http://icecast.streamserver.tld/mountpoint.aac': 'Another radio',
```
At least one source should be present if you use the `sources:` option, for units without any extra source input (like Audiocast M5) you should leave just `'wifi': 'WiFi'` if you don't want any webradios, that will actually hide the source switch as it's useless.

## Multiroom

Linkplay devices support multiroom in two modes:
- WiFi direct mode, where the master turns into a hidden AP and the slaves connect diretcly to that (legacy, below firmware v4.2.8020)
- Router mode, where the master and slaves connect to each other through the local network (faster but relying on your infrastructure).

This integration will autodetect the firmware version running on the player and choose multiroom mode accordingly. Firmware version number can be seen in device attributes. Can be overriden with option `multiroom_wifidirect: True` (see above).

To create a multiroom system, connect `media_player.sound_room2` (slave) to `media_player.sound_room1` (master):
```yaml
    - service: linkplay.join
      data:
        entity_id: media_player.sound_room2
        master: media_player.sound_room1
```
To exit from multiroom, use the entity ids of the players that need to be unjoined from the group. If this is the entity of a master, all slaves will be disconnected:
```yaml
    - service: linkplay.unjoin
      data:
        entity_id: media_player.sound_room1
```

## Recall device music presets

Linkplay devices allow to save, using the control app on the phone/tablet, music presets (for example Spotify playlists) to be recalled for later listening. Recalling a preset from Home Assistant:
```yaml
    - service: linkplay.preset
      data:
        entity_id: media_player.sound_room1
        preset: 1
```
Preset count vary from device tyoe, usually the app shows how many presets can be stored maximum. The integration detects the max number and the command only accepts numbers from the allowed range.

## Snapshot and restore

To save the player state:
```yaml
    - service: linkplay.snapshot
      data:
        entity_id: media_player.sound_room1
```
To restore the player state:
```yaml
    - service: linkplay.restore
      data:
        entity_id: media_player.sound_room1
```
Currently only the input source selection is being snapshotted/restored.

## Automation examples

To select an input and set volume and unmute via an automation:
```yaml
- alias: 'Switch to the line input of the TV when TV turns on'
  trigger:
    - platform: state
      entity_id: media_player.tv_room1
      to: 'on'
  action:
    - service: media_player.select_source
      data:
        entity_id: media_player.sound_room1
        source: 'TV sound'
    - service: media_player.volume_set
      data:
        entity_id: media_player.sound_room1
        volume_level: 1
    - service: media_player.volume_mute
      data:
        entity_id: media_player.sound_room1
        is_volume_muted: false
```
Note that you have to specify source names as you've set them in the configuration of the component.

To intrerupt playback of a source, say a TTS message and resume playback afterwards:
```yaml
- alias: 'Notify by TTS that Mary has arrived'
  trigger:
    - platform: state
      entity_id: person.mary
      to: 'home'
  action:
    - service: linkplay.snapshot
      data:
        entity_id: media_player.sound_room1
    - service: tts.google_translate_say
      data:
        entity_id: media_player.sound_room1
        message: 'Mary arrived home'
    - service: linkplay.restore
      data:
        entity_id: media_player.sound_room1
```


## Specific commands

Linkplay devices support some commands through the API, this is a wrapper to be able to use these in Home Assistant scripts of automations if needed:
```yaml
    - service: linkplay.command
      data:
        entity_id: media_player.sound_room1
        command: Reboot
```
Implemented commands:
- `Reboot` - as name implies
- `PromptEnable` and `PromptDisable` - enable or disable the audio prompts played through the speakers when connecting to the network or joining multiroom etc.
- `SetRandomWifiKey`- just as an extra security feature, one could make an automation to change the keys on the linkplay APs to some random values periodically.
- `WriteDeviceNameToUnit` - write the name configured in Home Assistant to the player’s firmware so that the names would match exactly in order to have multiroom in wifi direct mode fully operational (not needed for router mode)
- `TimeSync` - is for units on networks not connected to internet to compensate for an unreachable NTP server. Correct time is needed for the alarm clock functionality (not implemented yet here)
- `RouterMultiroomEnable` - theoretically router mode is enabled by default in firmwares above v4.2.8020, but there’s also a logic included to build it up, this command ensures to set the good priority. Only use if you have issues with multiroom in router mode.

Results will appear in Lovelace UI's left pane as persistent notifications which can be dismissed.


## About LinkPlay

LinkPlay is a smart audio chipset and module manufacturer. Their various module types share the same functionality across the whole platform and alow for native audio content playback from lots of sources, including local inputs, local files, Bluetooth, DNLA, Airplay and also web-based services like Icecast, Spotify, Tune-In, Deezer, Tidal etc. They allow setting up multiroom listening environments using either self-created wireless connections or relying on existing network infrastructure, for longer distances coverage. For more information visit https://linkplay.com/.
There are quite a few manufacturers and devices that operate on the basis of LinkPlay platform. Here are just some examples of the brands and models:
- **Arylic** (S50Pro, A50, Up2Stream),
- **August** (WS300G),
- **Audio Pro** (A10, A40, Addon C3/C5/C5A/C10/C-SUB, D-1, Drumfire, Link 1),
- **Auna** (Intelligence Tube),
- **Bauhn** (SoundMax 5),
- **Bem** (Speaker Big Mo),
- **Centaurus** (Flyears),
- **Champion** (AWF320),
- **COWIN** (DiDa, Thunder),
- **Crystal Acoustics** (Crystal Audio),
- **CVTE** (FD2140),
- **Dayton Audio** (AERO),
- **DOSS** (Deshi, Soundbox Mini, DOSS Assistant, Cloud Fox A1),
- **DYON** (DYON Area Player),
- **Edifier** (MA1),
- **Energy Sistem** (Multiroom Tower Wi-Fi, Multiroom Portable Wi-Fi),
- **FABRIQ** (Chorus, Riff),
- **First Alert** (Onelink Safe & Sound),
- **GE Sol** (C),
- **GGMM** (E2 Wireless, E3 Wireless, E5 Wireless),
- **GIEC** (Hi-Fi Smart Sound S1),
- **Harman Kardon** (Allure),
- **Hyundai** (Modern Oxygen Bar),
- **iDeaUSA** (iDEaHome, Home Speaker, Mini Home Soundbar),
- **iEAST Sonoé** (AudioCast M5, SoundStream, Stream Pro, StreamAmp AM160, StreamAmp i50B),
- **iHome** (iAVS16),
- **iLive** (Concierge, Platinum),
- **iLuv** (Aud Air, Aud Click Shower, Aud Click),
- **JAM Audio** (Voice, Symphony, Rhythm),
- **JD** (CrazyBoa 2Face),
- **Lowes** (Showbox),
- **Magnavox** (MSH315V),
- **Medion** (MD43631, MedionX MD43259),
- **Meidong** (Meidong 3119),
- **MK** (MK Alexa Speaker),
- **MÜZO** (Cobblestone),
- **Naxa** (NAS-5003, NHS-5002, NAS-5001, NAS-5000),
- **Nexum** (Memo),
- **Omaker** (WoW),
- **Omars** (Dogo),
- **Polaroid** (PWF1001),
- **Roxcore**	(Roxcore),
- **Sharper Image** (SWF1002),
- **Shenzhen Renqing Technology Ltd** (ROCKLAVA),
- **SoundBot** (SB600),
- **SoundLogic** (Buddy),
- **Stereoboommm** (MR200, MR300),
- **Tibo** (Choros Tap),
- **Tinman** (Smart JOJO),
- **Venz** (A501),
- **Uyesee** (AM160),
- **Youzhuan** (Intelligent Music Ceiling),
- **Zolo Audio** (Holo),
- etc.

## Home Assisnatnt component authors & contributors
    "@nicjo814",
    "@limych",
    "@nagyrobi"

## Home Assisnatnt component License

MIT License

- Copyright (c) 2019 Niclas Berglind nicjo814
- Copyright (c) 2019—2020 Andrey "Limych" Khrolenok
- Copyright (c) 2020 nagyrobi Robert Horvath-Arkosi

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

[forum-support]: https://community.home-assistant.io/t/linkplay-integration/33878