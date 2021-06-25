## `I` Preparation
> #### * Beginning of as root!
### `1` - Prepare the volume/partition
```sh
# Formatting.
mkfs.ext4 -m 0 -L "Heiwa_Linux" /dev/sdaX

# Export mountpoint variable and create the directory if not exists.
export HEIWA="/media/Heiwa"
mkdir -pv ${HEIWA}

# Mount the target volume/partition.
mount -vo noatime,discard /dev/sdaX ${HEIWA}
```
