obj-m   := fpga_push_switch_driver.o

#KDIR :=/work/achro-em/kernel
KDIR :=~/linux
PWD :=$(shell pwd)
KBUILD_EXTRA_SYMBOLS := /home/hwkwon/project/archro-em/fpga_example/fpga_interface_driver/Module.symvers

all: driver app

driver:
	make -C /lib/modules/4.19.106-v7l+/build M=$(shell pwd) modules
#	$(MAKE) -C $(KDIR) SUBDIRS=$(PWD) modules
app:
#	gcc -static -o fpga_test_push_switch fpga_test_push_switch.c
	arm-linux-gnueabihf-gcc -static -o fpga_test_push_switch fpga_test_push_switch.c

install_nfs:
	cp -a fpga_push_switch_driver.ko /nfsroot
	cp -a fpga_test_push_switch /nfsroot

install_scp:
	scp fpga_push_switch_driver.ko fpga_test_push_switch pi@192.168.0.xxx:/home/pi/Modules

clean:
	make -C /lib/modules/4.19.106-v7l+/build M=$(shell pwd) clean
#	$(MAKE) -C $(KDIR) SUBDIRS=$(PWD) clean
	rm -rf *.ko
	rm -rf *.mod.*
	rm -rf *.o
	rm -rf fpga_test_push_switch
	rm -rf Module.symvers
	rm -rf modules.order
	rm -rf .push_switch*
	rm -rf .tmp*

