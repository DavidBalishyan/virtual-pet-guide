# Session 9 - ESP-NOW Pet-to-Pet Communication (Bonus Feature)

Another bonus from the roadmap. This one needs two M5StickC Plus 2 devices - you can't test it with one. The idea is that two pets within WiFi range can exchange a "happiness gift" with each other. Shake your device and your pet sends a happiness boost to the other pet.

## What's planned

ESP-NOW is a protocol from Espressif that lets ESP32 devices communicate directly without going through a WiFi router. It works over the same radio hardware that WiFi uses but in a peer-to-peer mode. The devices discover each other by MAC address, send small packets of raw bytes, and receive them with a callback.

The planned PeerLink class would:
- Register each device's MAC address as a peer
- Send a PetGreeting struct (sender MAC + happiness value) when the device is shaken
- Receive greetings from other devices via a callback function
- Apply the received happiness boost to the pet

The PetGreeting struct would be simple:
```cpp
struct PetGreeting {
    uint8_t senderMac[6];  // 6 bytes - who sent this
    uint8_t happinessGift; // 1 byte - how much happy to give
};
```

The pairing model for the foundation is hardcoded: you look up the other device's MAC address (shown on its LCD) and type it into the source before flashing. A stretch goal would be automatic peer discovery through periodic broadcast packets.

For testing, shake device A and device B's happiness goes up, and vice versa. Both students see "the other pet said hi" on the screen. This is the highest-wow-factor bonus.
