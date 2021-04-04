wget http://ftp.br.debian.org/debian/pool/non-free/f/firmware-nonfree/firmware-bnx2x_20190114-2_all.deb
apt install ./firmware-bnx2x_20190114-2_all.deb


ADD the following to /etc/apt/sources.list
deb http://ftp.de.debian.org/debian buster main non-free
THEN run
apt update && apt install firmware-bnx2

Not all stuff is in the non-free so...
https://wiki.debian.org/DebianInstaller/Preseed/EditIso#Extracting_the_Initrd_from_an_ISO_Image

# Build the builder and then upload the iso to s3
```shell
# use this to include the preseed in the iso
export INCLUDE_PRESEED=1
ISO_NAME="custom_iso.iso"
PACKER_IMAGE="base-image.json"
PACKER_DIR="$PWD"
VAGRANT_DIR="$PACKER_DIR/../vagrant"
S3_BUCKET="your-s3-bucket"
if [[ "$INCLUDE_PRESEED" == "1" ]]; then
    ISO_NAME="custom_iso_with_preseed.iso"
    PACKER_IMAGE="base-image-preseed.json"
fi

# build the builder image
cd "$PACKER_DIR/builder-deb" && \
    packer build -only=vbox builder.json && \
    vagrant box add --force image-builder packer_output/builder_virtualbox.box

# use the builder image to create the custom iso
cd "$VAGRANT_DIR/builder-deb" && vagrant up

# upload the iso to s3
vagrant scp default:/tmp/custom_iso.iso "$PACKER_DIR/base-deb/$ISO_NAME"
aws s3 cp "$PACKER_DIR/base-deb/$ISO_NAME" "s3://$S3_BUCKET/$ISO_NAME" --acl public-read
ISO_CHKSUM=$(sha256sum "$PACKER_DIR/base-deb/$ISO_NAME" | cut -d' ' -f1)
```

# Recalculate checksum and build the base image locally (for testing)
```shell
cd $PACKER_DIR/base-deb && \
packer build \
    -only=vbox \
    -force \
    -var "iso_checksum=$ISO_CHKSUM" \
    -var "iso_url=$ISO_NAME" \
    "$PACKER_IMAGE"

vagrant box add --force base-image packer_output/packer_vbox_virtualbox.box
```
