# SimplePacketHandler

A PocketMine-MP library/virion that provides a simple API to handle data packets per network session.

[![CI](https://github.com/AxolotlBucketLabs/SimplePacketHandler/actions/workflows/test.yml/badge.svg)](https://github.com/AxolotlBucketLabs/SimplePacketHandler/actions/workflows/test.yml)
[![Latest release](https://img.shields.io/github/v/release/AxolotlBucketLabs/SimplePacketHandler)](https://github.com/AxolotlBucketLabs/SimplePacketHandler/releases/latest)
[![License](https://img.shields.io/github/license/AxolotlBucketLabs/SimplePacketHandler)](https://github.com/AxolotlBucketLabs/SimplePacketHandler/blob/pm5/LICENSE)
[![Discord](https://img.shields.io/discord/986553214889517088?label=Discord&logo=discord&logoColor=white&color=5865F2)](https://discord.gg/j2X83ujT6c)

## Compatibility

The virion manifest requires Axolotl-PM API `5.36.0` or newer. The Composer test environment pins Axolotl-PM `5.49.1`. See the [test workflow](https://github.com/AxolotlBucketLabs/SimplePacketHandler/actions/workflows/test.yml) for the exact checks.

## Installation

Use SimplePacketHandler as a virion in the consuming plugin. Put this repository in `virions/SimplePacketHandler/` and declare the manifest name in the consumer's `devtools.yml`:

```yaml
virions:
  - name: PacketHandler
    version: ^0.1.10
```

DevTools shares the source during development and injects the virion into the consumer PHAR during the build. Do not install this virion as a standalone plugin.

## API

### Packet monitor
Monitor data packets when you need to inspect them without changing the event outcome. The packet monitor registers receive and send handlers at `MONITOR` priority.

Use cases include the following:

- Dump extra data from packets.
- Debug whether a packet was sent or received.

```php
use muqsit\simplepackethandler\SimplePacketHandler;

/** @var Plugin $plugin */
$packet_monitor = SimplePacketHandler::createMonitor($plugin);

$packet_monitor->monitorIncoming(function(LoginPacket $packet, NetworkSession $origin) : void{
	$this->getLogger()->debug("Received LoginPacket from #" . spl_object_id($origin));
});

$packet_monitor->monitorIncoming(function(ServerSettingsResponsePacket $packet, NetworkSession $origin) : void{
	$this->getLogger()->debug("Received server settings response from {$origin->getPlayer()->getName()}");
});

$packet_monitor->monitorOutgoing(function(SetActorDataPacket $packet, NetworkSession $target) : void{
	$this->getLogger()->debug("Sent SetActorDataPacket to #" . spl_object_id($target));
});
```

The `LoginPacket` example above is the equivalent of:
```php
/**
 * @param DataPacketReceiveEvent $event
 * @priority MONITOR
 */
public function onDataPacketReceive(DataPacketReceiveEvent $event) : void{
	$packet = $event->getPacket();
	if($packet instanceof LoginPacket){
		$origin = $event->getOrigin();
		$this->getLogger()->debug("Received LoginPacket from #" . spl_object_id($origin));
	}
}
```

The `SetActorDataPacket` example above is the equivalent of:
```php
/**
 * @param DataPacketSendEvent $event
 * @priority MONITOR
 */
public function onDataPacketSend(DataPacketSendEvent $event) : void{
	foreach($event->getPackets() as $packet){
		if($packet instanceof SetActorDataPacket){
			foreach($event->getTargets() as $target){
				$this->getLogger()->debug("Sent SetActorDataPacket to #" . spl_object_id($target));
			}
		}
	}
}
```

### Packet interceptor
Handle data packets when you need to modify or cancel them. The default interceptor priority is `NORMAL`.

Use cases include the following:

- Block PocketMine-MP from handling specific data packets.
- Modify data packets before PocketMine-MP handles them.

```php
/** @var Plugin $plugin */
$packet_interceptor = SimplePacketHandler::createInterceptor($plugin);

$packet_interceptor->interceptIncoming(function(AdventureSettingsPacket $packet, NetworkSession $origin) : bool{
	if($packet->getFlag(AdventureSettingsPacket::FLYING)){
		return false; // cancels the DataPacketReceiveEvent
	}
	return true; // do nothing
});

$packet_interceptor->interceptOutgoing(function(SetTimePacket $packet, NetworkSession $target) : bool{
	$custom_player = CustomPlayerManager::get($target->getPlayer());
	if($custom_player->getPTime() !== $packet->time){
		$target->sendDataPacket(SetTimePacket::create($custom_player->getPTime()));
		return false;
	}
	return true;
});
```

The `AdventureSettingsPacket` example above is the equivalent of:
```php
/**
 * @param DataPacketReceiveEvent $event
 * @priority NORMAL
 */
public function onDataPacketReceive(DataPacketReceiveEvent $event) : void{
	$packet = $event->getPacket();
	if($packet instanceof AdventureSettingsPacket){
		$origin = $event->getOrigin();
		if($packet->getFlag(AdventureSettingsPacket::FLYING)){
			$event->cancel();
		}
	}
}
```

The `SetTimePacket` example above is the equivalent of:
```php
/**
 * @param DataPacketSendEvent $event
 * @priority NORMAL
 */
public function onDataPacketSend(DataPacketSendEvent $event) : void{
	foreach($event->getPackets() as $packet){
		if($packet instanceof SetTimePacket){
			$targets = $event->getTargets();
			$new_targets = $targets;
			foreach($new_targets as $index => $target){
				$custom_player = CustomPlayerManager::get($target->getPlayer());
				if($custom_player->getPTime() !== $packet->time){
					$target->sendDataPacket(SetTimePacket::create($custom_player->getPTime()));
					unset($new_targets[$index]);
				}
			}

			if(count($new_targets) !== count($targets)){
				// Cancel the event, try sending the remaining targets the
				// batch of packets again.
				$event->cancel();
				if(count($new_targets) > 0){
					$new_target_players = [];
					foreach($new_targets as $new_target){
						$new_target_players[] = $new_target->getPlayer();
					}
					$this->getServer()->broadcastPackets($new_target_players, $event->getPackets());
				}
				break;
			}
		}
	}
}
```
