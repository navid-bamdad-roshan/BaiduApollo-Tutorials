# Testing Apollo on Lexus Car

## General Information

In order to run the Apollo on the LEXUS RX 450h, we need some setups on the ubuntu operating system, and some changes have to be applied on the [official repo of Apollo](https://github.com/ApolloAuto/apollo) which are listed below.

## Setup SocketCAN on Ubuntu

- While the operating system is running inside the car, and the Kvaser is connected to the computer, run the following commands

  ```UNIX
  sudo modprobe can
  sudo modprobe can_raw
  sudo ip link set can0 type can bitrate 500000
  sudo ip link set up can0
  ```

## Changes to Apollo source code

### 1. Configuring the `canbus_conf.pb.txt` file

- The `modules/canbus/conf/canbus_conf.pb.txt` file should be as follows

  ```
  vehicle_parameter {
    brand: LEXUS
    max_enable_fail_attempt: 5
    driving_mode: COMPLETE_AUTO_DRIVE
  }

  can_card_parameter {
    brand: SOCKET_CAN_RAW
    type: USB_CARD
    channel_id: CHANNEL_ID_ZERO
    interface: NATIVE
  }

  enable_debug_mode: false
  enable_receiver_log: false
  enable_sender_log: false
  ```

### 2. Removing the filters on CAN messages

- Apollo tries to assign 2048 filters on can message IDs, but the problem is that linux kernel does not accept more than 512 filters. So the filter assigning part has to be deleted completely.
- Comment out the following code in `apollo/modules/drivers/canbus/can_client/socket/socket_can_client_raw.cc`

  ```cpp
  // 1. for non virtual busses, set receive message_id filter, ie white list
  if (interface_ != CANCardParameter::VIRTUAL) {
    struct can_filter filter[2048];
    for (int i = 0; i < 2048; ++i) {
      filter[i].can_id = 0x000 + i;
      filter[i].can_mask = CAN_SFF_MASK;
    }

    ret = setsockopt(dev_handler_, SOL_CAN_RAW, CAN_RAW_FILTER, &filter,
                    sizeof(filter));
    if (ret < 0) {
      AERROR << "add receive msg id filter error code: " << ret;
      return ErrorCode::CAN_CLIENT_ERROR_BASE;
    }
  }
  ```

### 3. Setting a lower default turning rate

- Default turning rate in Apollo for turning the steering wheel is to high in `lexus_controller.cc` (40 rad/sec) which cause the Pacmod to ignore the steering command. So, a lower turning rate has to be set to get it running using `canbus_teleop.sh`.
- In file `apollo/modules/canbus/vehicle/lexus/lexus_controller.cc`, method `void LexusController::Steer(double angle)` line `496`
  ```cpp
  void LexusController::Steer(double angle) {
    if (driving_mode() != Chassis::COMPLETE_AUTO_DRIVE &&
        driving_mode() != Chassis::AUTO_STEER_ONLY) {
      AINFO << "The current driving mode does not need to set steer.";
      return;
    }
    const double real_angle = vehicle_params_.max_steer_angle() * angle / 100.0;
    // TODO(Yu/QiL): double checck to decide if reverse sign needed
    steering_cmd_12c_->set_position(real_angle);
    // TODO(QiL) : double check this rate
    steering_cmd_12c_->set_rotation_rate(40);
  }
  ```
  change <br>`steering_cmd_12c_->set_rotation_rate(40);`<br> to <br>`steering_cmd_12c_->set_rotation_rate(2);`<br>

### 4. Changing the turning direction

- The steering wheel turns the wrong direction while using `canbus_teleop.sh`, so, the following change has to be applied.
- In file `apollo/modules/canbus/vehicle/lexus/protocol/steering_cmd_12c.cc`, method `void Steeringcmd12c::set_p_position(uint8_t* data, double position)`, line `129`

  ```cpp
  void Steeringcmd12c::set_p_position(uint8_t* data, double position) {
    position = ProtocolData::BoundedValue(-32.768, 32.767, position);
    int x = static_cast<int>(position / -0.001000);
    uint8_t t = 0;

    t = static_cast<uint8_t>(x & 0xFF);
    Byte to_set0(data + 2);
    to_set0.set_value(t, 0, 8);
    x >>= 8;

    t = static_cast<uint8_t>(x & 0xFF);
    Byte to_set1(data + 1);
    to_set1.set_value(t, 0, 8);
  }
  ```

  - Remove the `-` in this line<br> `int x = static_cast<int>(position / -0.001000);`

### 5. Re build the Apollo

- Launch the container `./docker/scripts/dev_start.sh`
- Enter the container `./docker/scripts/dev_into.sh`
- `./apollo.sh build_opt_gpu drivers/canbus/can_client/socket`
- `./apollo.sh build_opt_gpu canbus/vehicle/lexus`

## Setup GNSS

**Note:** This step is not required for driving the car using keyboard.<br>
**Note:** This gonfig file is for **NovAtel PwrPak7D-E2™ (Kit 2.5)** gnss receiver.
You have to replace the `<gnss receiver ip address>`, `<gnss receiver port number>`, and `<proj4_text for your region>` with correct values.

- The `modules/drivers/gnss/conf/gnss_conf.pb.txt` file should be as follows

  ```
  data {
      format: NOVATEL_BINARY
      tcp {
          address: "<gnss receiver ip address>"
          port: <gnss receiver port number>
      }
  }

  rtk_solution_type: RTK_RECEIVER_SOLUTION
  imu_type: ADIS16488
  proj4_text: "<proj4_text for your region>"


  # If given, the driver will send velocity info into novatel one time per second
  wheel_parameters: "SETWHEELPARAMETERS 100 1 1\r\n"
  #########################################################################
  # notice: only for debug, won't configure device through driver online!!!
  #########################################################################
  login_commands: "UNLOGALL THISPORT\r\n"
  login_commands: "LOG  GPRMC ONTIME 1.0 0.25\r\n"
  #login_commands: "EVENTOUTCONTROL MARK2 ENABLE POSITIVE 999999990 10\r\n"
  #login_commands: "EVENTOUTCONTROL MARK1 ENABLE POSITIVE 500000000 500000000\r\n"
  login_commands: "LOG GPGGA ONTIME 1.0\r\n"

  #login_commands: "log bestgnssposb ontime 1\r\n"
  #login_commands: "log bestgnssvelb ontime 1\r\n"
  #login_commands: "log bestposb ontime 1\r\n"
  #login_commands: "log INSPVAXB ontime 0.5\r\n"
  #login_commands: "log INSPVASB ontime 0.01\r\n"
  #login_commands: "log CORRIMUDATASB ontime 0.01\r\n"
  #login_commands: "log RAWIMUSXB onnew 0 0\r\n"
  #login_commands: "log INSCOVSB ontime 1\r\n"
  #login_commands: "log mark1pvab onnew\r\n"
  #
  #login_commands: "log rangeb ontime 0.2\r\n"
  #login_commands: "log bdsephemerisb\r\n"
  #login_commands: "log gpsephemb\r\n"
  #login_commands: "log gloephemerisb\r\n"
  #login_commands: "log bdsephemerisb ontime 15\r\n"
  #login_commands: "log gpsephemb ontime 15\r\n"
  #login_commands: "log gloephemerisb ontime 15\r\n"

  #login_commands: "log imutoantoffsetsb once\r\n"
  #login_commands: "log vehiclebodyrotationb onchanged\r\n"

  login_commands: "LOG GPRMC ONTIME 1.0 0.25\r\n"
  login_commands: "LOG GPGGA ONTIME 1.0\r\n"
  login_commands: "LOG BESTGNSSPOSB ONTIME 1\r\n"
  login_commands: "LOG BESTGNSSVELB ONTIME 1\r\n"
  login_commands: "LOG BESTPOSB ONTIME 1\r\n"
  login_commands: "LOG INSPVAXB ONTIME 1\r\n"
  login_commands: "LOG INSPVASB ONTIME 0.01\r\n"
  login_commands: "LOG CORRIMUDATASB ONTIME 0.01\r\n"
  login_commands: "LOG RAWIMUSXB ONNEW 0 0\r\n"
  login_commands: "LOG MARK1PVAB ONNEW\r\n"

  login_commands: "LOG RANGEB ONTIME 1\r\n"
  login_commands: "LOG BDSEPHEMERISB ONTIME 15\r\n"
  login_commands: "LOG GPSEPHEMB ONTIME 15\r\n"
  login_commands: "LOG GLOEPHEMERISB ONTIME 15\r\n"

  login_commands: "LOG INSCONFIGB ONCE\r\n"
  login_commands: "LOG VEHICLEBODYROTATIONB ONCHANGED\r\n"

  logout_commands: "EVENTOUTCONTROL MARK2 DISABLE\r\n"
  logout_commands: "EVENTOUTCONTROL MARK1 DISABLE\r\n"
  ```

## Drive car using keyboard

`canbus_teleop.sh` can be used to drive the car using keyboard. `canbus_teleop.sh` acts as a fake control module and publishes control messages. Then `canbus` module listens to them and communicate with car accordingly.

- Launch the container `./docker/scripts/dev_start.sh`
- Enter the container `./docker/scripts/dev_into.sh`
- Start canbus module `./scripts/canbus.sh start`
- Start canbus_teleop `./scripts/canbus_teleop.sh`
  - Press `m + 0` to reset car to manual mode
  - press `m + 1` to set cat to auto mode
  - press `w` to release brake / increase throttle
  - press `s` to push brake / decrease throttle
  - press `a/d` to control steering wheel
  - press `ctrl+c` then `enter` to exit

## Recording trajectory waypoints

- Setup and run rtk_recorder.sh

  ```
  ./scripts/rtk_recorder.sh setup
  ./scripts/rtk_recorder.sh start
  ```

- Drive around to record waypoints. Once you are finished, press `Ctrl + C` to stop recording.
- The recorded waypoints are stored in the `/apollo/data/log/garage.csv`

## Following recorded trajectory

- Drive the car to start location of trajectory recording.
- Run following to start rtk_player

  ```
  ./scripts/rtk_player.sh setup
  ./scripts/rtk_player.sh start
  ```

- Run `./bazel-bin/modules/control/tools/pad_terminal` and press `1` to enter auto driving mode. By entering auto driving mode, apollo will control the car to follow the recorded path.

  **Note**

  `modules/control/tools/pad_terminal.cc` is used to set driving mode `COMPLETE_MANUAL` / `COMPLETE_AUTO_DRIVE`
