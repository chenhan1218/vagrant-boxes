export VAGRANT_DEFAULT_PROVIDER = virtualbox

DISTRIBUTIONS = $(basename $(wildcard *.json))
TESTED = $(addsuffix .tested, $(DISTRIBUTIONS))
SHELL = /bin/bash # needed for |&
# exit on undefined variables, pipeline fails if one command fails
.SHELLFLAGS = -u -o pipefail -c
RELEASE_NUMERIC = $(shell jq --raw-output '.variables.box_version' $*.json )
TEE = |& ts |& tee --append $*.log # log every command

all:
	@echo "Usage: make <distribution>"
	@echo "Available distributions: $(DISTRIBUTIONS)"

include ../helpers/vagrantBoxTesting.mk

.PRECIOUS: %.build %.vmdk %.box

$(DISTRIBUTIONS): % : %.box
$(TESTED): % :
test-all: $(TESTED)

# set the shared folder method to rsync for most boxes
VAGRANT_FILE = Vagrantfile.in
# contrib boxes use VirtualBox default, ie vboxfs kernel module
contrib-%.box: VAGRANT_FILE= Vagrantfile-contrib.in

%.box: %.vmdk
	scripts/import2vbox.pl --memory 512 --vcpus 2 $*.vmdk $(TEE)
	mkdir -p $*.box.in
	cp $(VAGRANT_FILE) $*.box.in/Vagrantfile
	cp $*.ovf $*.box.in/box.ovf
	cp $*.vmdk $*.box.in/
	echo '{"provider": "virtualbox"}' > $*.box.in/metadata.json
	cd $*.box.in && tar czvf ../$*.box ./metadata.json ./box.ovf ./$*.vmdk ./Vagrantfile

%.vmdk: %.build
	qemu-img convert -O vmdk $</$* $*.vmdk $(TEE)

%.build:
	packer build $*.json |& ts |& tee $*.log # create a new log file

# test initialization / login / network
%.tested: %.box
	../helpers/vagrant-test virtualbox $* $< $(TEE)
	touch $@ $(TEE)

testing.uploaded: ISO_DATE_SEMVER = --version $(shell date +%Y.%m.%d)
%.uploaded: $(VAGRANT_DEFAULT_PROVIDER)-%.SHA256SUM.gpg
	../helpers/vagrant-cloud.pl --box $*.box $(ISO_DATE_SEMVER)
	@echo "everything done, please release the box $*64 on app.vagrantup.com"
	touch $@

$(VAGRANT_DEFAULT_PROVIDER)-%.SHA256SUM: %.tested
	sha256sum $*.box > $@
	tr '\n' ' ' < $@ | sponge $@
	echo $(RELEASE_NUMERIC) >> $@

$(VAGRANT_DEFAULT_PROVIDER)-%.SHA256SUM.gpg: $(VAGRANT_DEFAULT_PROVIDER)-%.SHA256SUM
	gpg --batch --yes --sign $<

.PRECIOUS: $(VAGRANT_DEFAULT_PROVIDER)-%.SHA256SUM $(VAGRANT_DEFAULT_PROVIDER)-%.SHA256SUM.gpg
.PHONY: clean cleanall cleanrelease %.cleantest

clean::
	-$(RM) *.build/*
	-rmdir *.build/
	-$(RM) *.box.in/*
	-rmdir *.box.in/
	-$(RM) *.vmdk
	-$(RM) *.ovf
	-$(RM) *.SHA256SUM.gpg

cleanall: clean
	$(RM) *.box *.tested *.uploaded *.released
