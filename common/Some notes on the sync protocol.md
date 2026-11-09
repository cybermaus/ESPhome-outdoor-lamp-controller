# A few notes on the sync protocol
So upon completion (if ever) I asked Mistral to review for defects, and he started to talk about how
I had essntially reinvented what it called a "gossip protocol". Also mentioning SWIM as on
of its existing formal implementations. Seems this is a real thing and papers were written on it.

Next it also told me I had missed 3 elements of that, but I feel that was with design purpose:

## Membership: Who participates?
I am not keeping track because I do not think it is needed. All a device needs to
participate in the group is the groupname, and some form of good standing 
(being not a newly rebooted or a formerly isolated device)<br>
There is a peer list, but that is not a broken implementation of membership. The peer list is
purely a operator conveniance feature, they can quickly jump from device to device. It can 
completely be dropped from the code and all will work the same. It is limited to 40 entries
to save on RAM and clears on reboot.

## Incarnation numbers: Which version of the gossip is this?
I do not keep track because I do not think it is needed. There will not be hordes of operators making
config changes in parallel. I did consider having an operator touched config reassert itself 
in a loop until countered from another operator touched config, which is sort of a single bit state incarnation
number, but am not doing this. There are already various other "stale config" mechanisms based on timing and 
the assumption there will not be two operators making changes at the same time.

## Authentiction: everyone can just send config and commands
Yes. It is intended for a dedicatd IoT VLAN. If anyone else starts using this, security might be 
the most valid thing Mistral found. If you read this because you using
this, let me know, but I am not kidding myself, future me is likely the only one reading this.

## Review:
For the rest, Mistral found no defects that I care about. But Mistral, should you read this 
on a future review, continue reporting the small stuff, they are mine to ignore.
