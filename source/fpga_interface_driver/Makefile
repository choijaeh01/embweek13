#Makefile for a basic kernel module
#export PATH=$PATH:~/tools/arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian-x64/bin

obj-m   := fpga_interface_driver.o

KDIR :=/work/achro-em/kernel/
#KDIR :=~/linux
PWD :=$(shell pwd)
all: driver

driver:
	make -C /lib/modules/4.19.106-v7l+/build M=$(shell pwd) modules
#	$(MAKE) -C $(KDIR) SUBDIRS=$(PWD) modules

install_nfs:
	cp -a fpga_interface_driver.ko /nfsroot

install_scp:
	scp fpga_interface_driver.ko pi@192.168.0.120:/home/pi/Modules

clean:
	make -C /lib/modules/4.19.106-v7l+/build M=$(shell pwd) clean
#	$(MAKE) -C $(KDIR) SUBDIRS=$(PWD) cleanclean:
	rm -rf *.ko
	rm -rf *.mod.*
	rm -rf *.o
	rm -rf Module.symvers
	rm -rf modules.order
	rm -rf .interface*
	rm -rf .tmp*
