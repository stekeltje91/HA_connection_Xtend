# HA_connection_Xtend

An HowTo article on connecting the Intergas Xtend hybrid heatpump to Home Assistant (HA) via wifi network adapter (multi homed) in three steps:

1. Connect HA to Xtend via Wifi
1. Install the Xtend package in HA
1. Create a Xtend Dashboard

Hope it helps it in your journey.

> [!NOTE]
> This is a fork of [DSchoutsen/HA_connection_Xtend](https://github.com/DSchoutsen/HA_connection_Xtend), updated and tested for **Home Assistant 2026.10**. All credits for the original work and the Xtend field research go to Dennis Schoutsen.
>
> **Changes in this fork**
>
> * **One package file** ([config/packages/intergas_xtend.yaml](config/packages/intergas_xtend.yaml)) instead of separate sensor and template files.
> * **One API request** for heat pump *and* boiler data. The original polled the Xtend twice (`Intergas_Xtend` + `Intergas_Xtreme`), which adds load on the Xtend.
> * **Fixed: boiler gas meter** (`xtreme_gas_meter_ch`) never had data, because field `7191` was not requested.
> * **Fixed: field `848e`** (boiler modulation level set) is no longer requested. It is known to break the API connection (upstream issue #9), and was still in the boiler request. The sensor `xtreme_boiler_ot_modulation_level_set` is removed.
> * **Availability templates on every sensor.** When the Xtend Wi-Fi drops, sensors become *unavailable* instead of filling the log with template errors. Energy counters can no longer drop to 0 and create false spikes in statistics.
> * **32767** (the Xtend's "no value" for 16-bit fields) is shown as *unavailable* instead of e.g. 327.67 °C.
> * **Fixed: Delta T** was rounded down to whole degrees.
> * **Correct units and device classes**: `kW` instead of `KW`, `Hz` instead of `hertz`, pressure/flow/frequency/power classes, and `state_class` on all measurements for long-term statistics.
> * **Energy dashboard ready**: electric and thermal energy use device class `energy`, the boiler gas meter uses device class `gas`.
> * **Dashboards without HACS**: only built-in cards (tile, sensor, gauge, heading), and the alert tiles now actually appear when the Xtend reports a notification, error or lockout.
>
> Entity IDs and unique IDs are unchanged, so existing dashboards and history keep working.

## Connect HA to Xend via Wifi

The image below illustrates my setup at home. Your setup maybe different but the concepts still apply.

![xtend_setup](images/xtend_setup.jpg)

The device where HA is running needs to have at least two network devices:

1. _Ethernet adapter_ - recommended for default HA communication (see [HA installation doc](https://www.home-assistant.io/installation/raspberrypi#:~:text=Home%20Assistant%20can%20work%20with%20Wi%2DFi%2C%20but%20an%20Ethernet%20connection%20is%20more%20reliable%20and%20highly%20recommended.))
1. _WLAN adapter_ - to connect to the Xtend Wifi Access point.

In HA network adapters are automatically discovered and available in _Settings / System / Network_.

![HA_network](images/HA_network.jpg)

> [!NOTE]  
>
> * Make sure Home Assistant and the Xtend unit are within Wifi range of each other.
> * The IP-range of your LAN cannot overlap with 10.20.30.0/24
> * I disabled the auto-config on the HA Network Adapter section, because i suspect it from disabling the IP4 configuration of my wifi adapter. The configuration in the image below provides me with a stable connection with the Xtend.
> ![HA Network adapter](images/HA_networkAdapter.jpg)

Follow these steps to connect HA to the Xtend:

* Press the button on the Xtend to activate the Wifi Access point (the LED blinks purple)
* In HA click on _Scan for Access points_, Select the network _Xtend_xxxxxxxxxx_
* Select _wpa-psk_ and typ the password (read the Xtend documentation where to locate your password)

  ![HA_wifipassword](images/HA_wifipassword.jpg)

* Click _Save_
* When connected succesfully the IP-information (behind the three dots) should look like this.

  ![HA_wifiIpInformation](images/HA_wifiIpInformation.jpg)

> [!TIP]
> The Xtend Wi-Fi network has no internet. Make sure the Ethernet adapter stays the default route, otherwise other integrations may lose internet access (upstream issue #8). Do not set a gateway or DNS server on the Wi-Fi adapter.

## Install the Xtend package in HA

All Xtend configuration is in one [Home Assistant package](https://www.home-assistant.io/docs/configuration/packages/): [config/packages/intergas_xtend.yaml](config/packages/intergas_xtend.yaml). It contains:

* one REST sensor `sensor.intergas_xtend` that reads all fields from the Xtend API every 2 minutes, and stores them in its `stats` attribute;
* template sensors that turn those raw values into separate entities (`sensor.xtend_*` for the heat pump, `sensor.xtreme_*` for the OpenTherm boiler data);
* two binary sensors that show whether the heat pump and the boiler are running.

Follow these steps:

1. Enable packages. Add this to your _configuration.yaml_ (if you already have a `homeassistant:` section, add only the `packages:` line to it):

   ```yml
   homeassistant:
     packages: !include_dir_named packages
   ```

1. Create the folder _/config/packages/_ and copy [intergas_xtend.yaml](config/packages/intergas_xtend.yaml) into it. The [VS Code Server](https://github.com/hassio-addons/addon-vscode) or [File editor](https://github.com/home-assistant/addons/tree/master/configurator) add-on makes this easy.
1. In HA open _Developer Tools_ / _YAML_ and click on _Check configuration_.
1. When it reports _Configuration will not prevent Home Assistant from starting!_, click _Restart_.
1. In _Developer Tools_ / _States_ filter on _intergas_xtend_. The state is `Ok`, and the `stats` attribute shows the values from the Xtend:

   ![States in Developer Tools ](images/HA_DevelopertoolsStates.jpg)

1. Filter on _xtend__ and _xtreme__ to see all the new entities.

   ![image](images/HA_entities.jpg)

> [!NOTE]
>
> * **Coming from the original split files?** First remove _sensors/sensor_intergas_Xtend.yaml_, _sensors/sensor_intergas_xtreme.yaml_, _templates/template_intergas_Xtend.yaml_ and _templates/template_intergas_Xtreme.yaml_. The unique IDs are the same, so your entities and their history are kept. Afterwards delete the old `sensor.intergas_xtreme` entity in _Settings / Entities_.
> * **No OpenTherm boiler data** (e.g. some Kompakt HRE boilers): remove the fields `623c,625b,7191,7e7a,8434,844c,84d1,8e00,8e18,8e1e,8e37,8e7f,8e8f,8ecb,8ecc,8edb,8ef9` from the `resource` URL, and the sensors under _Boiler (OpenTherm) data_. Otherwise those sensors simply stay unavailable.
> * There are a lot more fields available from the Xtend API. See the file [Known Xtend Codes](Known_Xtend_codes.md) or [parse-message.js](https://github.com/thomasvt1/xtend-bridge/blob/main/parse-message.js) from thomasvt1. Request only the fields you use: more fields means more load on the Xtend.

> [!TIP]
> Don't configure a scan interval above 300 (5 min). In my experience the wifi connection will disconnect due to inactivity and you need to repeat the previous paragraph, including climbing the stairs to push the button on the Xtend.
> Don't configure it too low as well or it will flood your HA database and (maybe) the storage capacity of your HA device.

### Troubleshooting

* **All Xtend sensors are unavailable**: check the Wi-Fi connection to the Xtend first. From the VS Code Server / Terminal add-on, run `curl "http://10.20.30.1/api/stats/values?fields=47e0"`. It should return the software version.
* **Sensors drop out after a while**: some users reported fields that break the connection on their Xtend (for example `5077` and `6505`, upstream issue #9). Remove a field from the `resource` URL to test. Power-cycling the Xtend (30 seconds) also helps.
* **Database size**: `sensor.intergas_xtend` stores all raw values as an attribute every 2 minutes. If you don't need its history, exclude it from the [recorder](https://www.home-assistant.io/integrations/recorder/):

  ```yml
  recorder:
    exclude:
      entities:
        - sensor.intergas_xtend
  ```

### Energy dashboard

In _Settings / Dashboards / Energy_ you can add:

* `sensor.xtend_electricenergyheating` as an individual device (electricity used by the heat pump);
* `sensor.xtreme_gas_meter_ch` under gas consumption (gas used by the boiler for central heating).

## Create a Xtend Dashboard

So far so good .... it's time to use the obtained data in a dashboard.

> [!NOTE]
>
> * This example dashboard is inspired by the Statistics page on the Xtend webinterface. As noted before it is possible to retrieve other fields from the Xtend API and present these on the Xtend dashboard.
> * The dashboards use the Sections view and only built-in cards. No HACS cards are needed.
> * The Xtend provide a weird value of **32767** (maximum Integer value) for pressures and temperatures. These sensors show as _unavailable_ in that case.

There are two dashboards. One with [Xtend heat pump and OpenTherm boiler data](Xtend_boiler_dashboard.yaml) and one with only the [Xtend heatpump data](Xtend_dashboard.yaml). The notification, error and lockout tiles only appear when the Xtend reports something.

![XtendDash](images/HA_Xtend_boiler_dashboard.png)

> [!NOTE]
>
> * This screenshot shows the original dashboard. The layout of the updated dashboard is similar, with built-in tiles instead of big number cards.
> * The screenshot was taken just after the Xtend was installed in Summer!. The CoP of 13 is a result of some test runs and not a realistic value. :sunglasses:

Follow these steps to import this dashboard in your HA.

* Open the Overview page in HA and click on the pencil in the top right corner (Edit Dashboard)
* Click on the plus-sign to add a new view
* In the new window _View Configuration_ click on the three-dot menu in the top right corner and select _Edit in YAML_
  ![Edit in yaml](images/HA_newdashyaml.png)
* Copy the text from [Xtend_dashboard.yaml](Xtend_dashboard.yaml) or [Xtend_boiler_dashboard.yaml](Xtend_boiler_dashboard.yaml) and Paste the yaml code in the _View configuration_ Window
  ![image](images/HA_DashboardYaml.png)
* Click on _Save_
* Click on _Done_

I hope this article has helped you monitor your Xtend's performance in a convenient way in Home Assistant.

## Background

In july 2024 my Intergas Xtend Hybrid Heat pump was installed. In anticipation off the installation I did some research on the internet on how I could connect the Xtend to Home Assistant. The solution described in [Xtend_bridge by thomasvt1](https://github.com/thomasvt1/xtend-bridge) didn't appeal to me because of the added devices and cables required. That said the code in that repository provided a lot of information I used to configure my own, more HA native, setup.

I am by no means a HA expert and gather and borrow information from the internet. This solution may not be as elegant but I lack the skills to develop a proper HA Integration; As the saying goes: _Make do with what you have_
Recommendations and/or improvements are welcome.

### My HA setup

I run HA on a refurbished Lenovo Thinkcenter (i3-4160T/8GB/256SSD/W10) with an external USB Wifi adapter. I removed the Windows OS and installed it with HAOS according to this article.
[HA generic-x86-64 Install](https://www.home-assistant.io/installation/generic-x86-64).

For the adjusting the config files in HA I installed the [VS Code Server](https://github.com/hassio-addons/addon-vscode) Add-On.
