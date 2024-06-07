### Lower bound fee rate for `SoulFeeManager::getFee()` is incorrect ###

### Description ###
The implementation logic does not consider the lower bound scenario.

Example, lets say, the configured threshold values are as below.

[{ volume: 5000, fee:5 },{ volume: 6000, fee:6 },{ volume: 7000, fee:7 },{ volume: 8000, fee:8 },{ volume: 9000, fee:9 } ]

a) check for input Volume = 9200
Since Input Volume is greater than 9000, fee applicable is 9

b) check for input volume = 8999
Since the input volume is greater than 8000, fee applicable is 8

c) check for input volume = 4999
Since the input volume is not greater than any configured volume, the fee applicable is 0

This gap could arise due to human error during configuration.

### Recommendation ###
The suggestion is to apply the fee for the lowest index incase the volume is lower than the least configured volume threshold.

Suggested fix is as below where instead of returning 0, return the lowest configured fee.

```
function getFee(uint256 epochFeeVolume) public view returns (uint256 fee) {
for (uint256 i = volumeFeeThresholds.length - 1; i >= 0; i–) {
if (epochFeeVolume >= volumeFeeThresholds[i].volume) {
return volumeFeeThresholds[i].fee;
}
}
return volumeFeeThresholds[0].fee;
}```