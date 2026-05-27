obj-m   := fpga_dot_driver.o

KDIR :=/work/achro-em/kernel
PWD :=$(shell pwd)
KBUILD_EXTRA_SYMBOLS := /home/hwkwon/project/archro-em/fpga_example/fpga_interface_driver/Module.symvers

all: driver app

driver:
	make -C /lib/modules/4.19.106-v7l+/build M=$(shell pwd) modules
#	$(MAKE) -C $(KDIR) SUBDIRS=$(PWD) modules
app:
#	gcc -static -o fpga_test_dot fpga_test_dot.c
	arm-linux-gnueabihf-gcc -static -o fpga_test_dot fpga_test_dot.c

install_nfs:
	cp -a fpga_dot_driver.ko /nfsroot
	cp -a fpga_test_dot /nfsroot

install_scp:
	scp fpga_dot_driver.ko fpga_test_dot pi@192.168.0.xxx:/home/pi/Modules

clean:
	make -C /lib/modules/4.19.106-v7l+/build M=$(shell pwd) clean
#	$(MAKE) -C $(KDIR) SUBDIRS=$(PWD) clean
	rm -rf *.ko
	rm -rf *.mod.*
	rm -rf *.o
	rm -rf fpga_test_dot
	rm -rf Module.symvers
	rm -rf modules.order
	rm -rf .dot*
	rm -rf .tmp*
