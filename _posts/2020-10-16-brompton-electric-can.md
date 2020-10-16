---
layout: post
title: "Brompton Electric CAN"
---
I recently got an [electric brompton](https://www.brompton.com/bikes/brompton-electric) that allowed me to commute the 16km journey to and from work quite comfortably, every day in London.

It has a claimed range of 40 to 80km, which I found to be around 50km on my daily commutes. The longest ride I did on it was a 100km trip from Central London to Box Hill in Surrey, where I chose to conserve my battery for the three major hill climbs (I got strange looks from riders on road bikes). This required me to learn to cycle just faster than maximum assist speed of 25kph by feeling the point when the electric motor will kick in, and with a quick lunchtime 20% SoC top up at a pub, I was able to make it all the way home.

The electric motor is developed by Williams Advanced Engineering and the battery made by BMZ ([material data sheet](https://en.calameo.com/read/0011021664128ccf06e0f)).

{% include image.html url="/assets/images/IMG_3021.jpeg" description="18650 NMC cells are used in a 10S3P configuration" %}

{% include image.html url="/assets/images/IMG_3018.jpeg" description="An EnergyBus Connector is used, although the CAN messages do not follow the EnergyBus specifications" %}

{% include image.html url="/assets/images/IMG_2928.jpeg" description="Charging port with CAN high (red) and CAN low wires connected to a Vector CAN dongle or Raspberry Pi" %}

The electronic components (battery, motor, pedal sensors, controller) communicate via CAN at 500kbps. The busload was 35%, which is pretty high for a bicycle - maybe this was a Williams design decision. To record data during a cycling trip, I attached a Raspberry Pi Zero and [CAN Hat](https://thepihut.com/products/rs485-can-hat-for-raspberry-pi) to the CAN lines of the charging port, and powered it from the built-in USB charger. 

I initially tried to use the `python-can` library to log the messages directly as a .blf file, but the overhead from python maxed out the Pi, and caused messages to drop. So I chose to use `candump` to record the data, and converted it to a .blf file offline.
```bash
$ cat /etc/systemd/system/candump.service
[Unit]
Description=candump
BindsTo=sys-subsystem-net-devices-can0.device
After=sys-subsystem-net-devices-can0.device

[Service]
User=pi
Group=pi
Restart=always
RestartSec=3
WorkingDirectory=/home/pi/candumps
ExecStartPre=/bin/mkdir -p /home/pi/candumps
ExecStart=/usr/bin/candump -l -D can0

[Install]
WantedBy=multi-user.target
```

There was a total of 33 different CAN IDs and by using CANalyzer, I was able to reverse engineer some of the messages and put together a dbc file. I'm sure some of the messages are interpreted or scaled wrongly, maybe someone at brompton can lend me the actual file!

System Controller:
```
BO_ 513 sys_201: 8 Vector__XXX
 SG_ sys_power_cmd : 0|32@1+ (1,0) [0|0] "" Vector__XXX
 SG_ sys_uptime : 32|32@1+ (0.008,0) [0|0] "s" Vector__XXX

BO_ 1537 wheel_601_cycling_speed_and_cadence: 8 Vector__XXX
 SG_ cycling_speed : 0|32@1- (1,0) [0|0] "kph" Vector__XXX
 SG_ cadence : 32|32@1- (1,0) [0|0] "rpm" Vector__XXX

SIG_VALTYPE_ 1537 cycling_speed : 1;
SIG_VALTYPE_ 1537 cadence : 1;
```

Wheel:
```
BO_ 514 wheel_202_sensor: 8 Vector__XXX
 SG_ wheel_202_sensor_pwr_cmd : 0|16@1+ (1,0) [0|0] "" Vector__XXX
 SG_ wheel_202_sensor_power : 32|16@1- (0.01,0) [0|0] "" Vector__XXX
 SG_ wheel_202_sensor_speed : 48|16@1+ (1,0) [0|0] "" Vector__XXX

BO_ 1538 wheel_602_elec: 8 Vector__XXX
 SG_ wheel_602_elec_current_a : 0|16@1- (1,0) [-32768|32767] "" Vector__XXX
 SG_ wheel_602_elec_current_b : 16|16@1- (1,0) [-32768|32767] "" Vector__XXX
 SG_ wheel_602_elec_current_c : 32|16@1- (1,0) [-32768|32767] "" Vector__XXX
 SG_ wheel_602_elec_voltage : 48|16@1+ (0.005,0) [0|0] "" Vector__XXX

BO_ 1541 wheel_605: 8 Vector__XXX
 SG_ wheel_605_sys_power_meas : 16|16@1- (1,0) [0|0] "" Vector__XXX
 SG_ wheel_605_sys_power_cmd : 48|16@1- (1,0) [0|0] "" Vector__XXX
```

Pedal Sensor:
```
BO_ 1547 pedal_sensor_60b: 8 Vector__XXX
 SG_ pedal_sensor_60b_phase : 0|8@1+ (1,0) [0|0] "" Vector__XXX
 SG_ pedal_sensor_60b_phase_filt : 16|8@1+ (1,0) [0|0] "" Vector__XXX
 SG_ pedal_sensor_60b_direction : 24|8@1+ (1,0) [0|0] "" Vector__XXX
 SG_ pedal_sensor_60b_hall_1 : 32|8@1+ (1,0) [0|0] "" Vector__XXX
 SG_ pedal_sensor_60b_hall_2 : 48|8@1+ (1,0) [0|0] "" Vector__XXX
```

Battery:
```
BO_ 1032 batt_408_pv: 8 Vector__XXX
 SG_ batt_408_pv_nominal_voltage : 0|16@1+ (1,0) [0|0] "V" Vector__XXX
 SG_ batt_408_pv_pack_voltage : 32|32@1+ (1E-006,0) [0|0] "V" Vector__XXX

BO_ 1033 batt_409_status: 8 Vector__XXX
 SG_ batt_409_status_current : 0|32@1- (0.005,0) [0|0] "A" Vector__XXX

BO_ 1034 batt_40a_energy: 8 Vector__XXX
 SG_ batt_40a_nominal_capacity : 0|16@1+ (0.001,0) [0|0] "Wh" Vector__XXX
 SG_ batt_40a_energy_wh_remaining : 40|16@1+ (1,0) [0|0] "Wh" Vector__XXX

BO_ 769 temp_301: 8 Vector__XXX
 SG_ temp_301_temp_1 : 8|8@1+ (1,0) [0|0] "degC" Vector__XXX
 SG_ temp_301_temp_2 : 16|8@1+ (1,0) [0|0] "degC" Vector__XXX
 SG_ temp_301_temp_3 : 32|8@1+ (1,0) [0|0] "degC" Vector__XXX
 SG_ temp_301_temp_state : 40|8@1+ (1,0) [0|0] "" Vector__XXX
```

From the decoded messages, I was able to create a live dashboard on my phone using a rust webserver running on the Pi (faster than python!) that streamed the data to the phone via a websocket. I was also able to override the command messages to command the motor, with the intention of bypassing the speed limit, which I will detail in another post and github repo once the code has been cleaned up.