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

## **TODO**

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

## **TODO**

## Following recorded trajectory

## **TODO**

**NB!**

Use `modules/control/tools/pad_terminal.cc` to set driving mode to `COMPLETE_MANUAL` / `COMPLETE_AUTO_DRIVE`
