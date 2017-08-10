# udev rule : SUBSYSTEMS=="usb", ATTRS{idVendor}=="0483", ATTRS{idProduct}=="5740", MODE="0666", SYMLINK+="lsm303agr"

import serial
import re
import time
import signal
import math

# registers

CFG_REG_A_M = "60" # comp_temp_en, reboot, soft_rst, lp, odr1, odr0, md1, md0
CFG_REG_B_M = "61" # 0,0,0, off_canc_one_shot, int_on_dataoff, set_freq, off_canc, lpf
CFG_REG_C_M = "62" # 0, int_mag_pin, i2c_dis, bdu, ble, self_test, int_mag

CTRL_REG1_A = "20" # odr3, odr2, odr1, odr0, lpen, zen, yen, xen 
CTRL_REG4_A = "23" # bdu, ble, fs1, fs0, hr, st1, st0, spi_enable

TEMP_CFG_REG_A = "1F" # temp_en1, temp_en0, 0,0,0,0,0,0

OUTX_L_REG_M = "68"
OUTX_H_REG_M = "69"
OUTY_L_REG_M = "6A"
OUTY_H_REG_M = "6B"
OUTZ_L_REG_M = "6C"
OUTZ_H_REG_M = "6D"

OUT_X_L_A = "28"
OUT_X_H_A = "29"
OUT_Y_L_A = "2A"
OUT_Y_H_A = "2B"
OUT_Z_L_A = "2C"
OUT_Z_H_A = "2D"

OFFSET_X_REG_L_M = "45"
OFFSET_X_REG_H_M = "46"
OFFSET_Y_REG_L_M = "47"
OFFSET_Y_REG_H_M = "48"
OFFSET_Z_REG_L_M = "49"
OFFSET_Z_REG_H_M = "4A"

OUT_TEMP_L_A = "0C"
OUT_TEMP_H_A = "0D"

_RUN_ = True
_LOWPOWER_ = False
_CALLIBRATE_ = True

def signal_handler(signal, frame):
	global _RUN_
	_RUN_ = False
	print("stop")

signal.signal(signal.SIGINT, signal_handler)

def twocplm2int(val, bits=16):
	if val>>(bits-1) != 0: val -= 1 << bits
	return val

def int2twocplm(val, bits=16):
	mask = (1 << bits-1)
	if val >= 0:
		return val

	return mask|(mask+val)

def regmwrite(addr, data):
	s.write(str.encode("*mw"+addr+data+"\r\n"))
	
def regmread(addr):
	s.write(str.encode("*mr"+addr+"\r\n"))
	return s.readline()

def regawrite(addr, data):
	s.write(str.encode("*w"+addr+data+"\r\n"))
	
def regaread(addr):
	s.write(str.encode("*r"+addr+"\r\n"))
	return s.readline()

def getmcoord(coord):

	if coord == "x":
		addrl = OUTX_L_REG_M
		addrh = OUTX_H_REG_M
	elif coord == "y":
		addrl = OUTY_L_REG_M
		addrh = OUTY_H_REG_M
	elif coord == "z":
		addrl = OUTZ_L_REG_M
		addrh = OUTZ_H_REG_M

	s.write(str.encode("*mr"+addrl+"\r\n"))
	l = s.readline()

	s.write(str.encode("*mr"+addrh+"\r\n"))
	h = s.readline()

	l = re.search("MR"+addrl+"h(\S{2})h\r\n", bytes.decode(l))[1]
	h = re.search("MR"+addrh+"h(\S{2})h\r\n", bytes.decode(h))[1]

	return twocplm2int( int(h+l,16) ) # concat MSB and LSB then convert

def getacoord(coord):

	if coord == "x":
		addrl = OUT_X_L_A
		addrh = OUT_X_H_A
	elif coord == "y":
		addrl = OUT_Y_L_A
		addrh = OUT_Y_H_A
	elif coord == "z":
		addrl = OUT_Z_L_A
		addrh = OUT_Z_H_A

	s.write(str.encode("*r"+addrl+"\r\n"))
	l = s.readline()

	s.write(str.encode("*r"+addrh+"\r\n"))
	h = s.readline()

	l = re.search("R"+addrl+"h(\S{2})h\r\n", bytes.decode(l))[1]
	h = re.search("R"+addrh+"h(\S{2})h\r\n", bytes.decode(h))[1]

	if not _LOWPOWER_:
		return twocplm2int( int(h+l,16)>>4 , 12) # concat MSB and LSB then convert
	else: # low power mode -> 8 bit resolution stored in MSB
		return twocplm2int( int(h,16), 8 )

def gettemp():
	s.write(str.encode("*r"+OUT_TEMP_L_A+"\r\n"))
	l = s.readline()

	s.write(str.encode("*r"+OUT_TEMP_H_A+"\r\n"))
	h = s.readline()

	l = re.search("R"+OUT_TEMP_L_A+"h(\S{2})h\r\n", bytes.decode(l))[1]
	h = re.search("R"+OUT_TEMP_H_A+"h(\S{2})h\r\n", bytes.decode(h))[1]

	# todo fix temp reading
	print(h+l)

	return twocplm2int( int(h,16)>>3, 8 )

s = serial.Serial("/dev/lsm303agr", 115200, timeout=1)

#init
s.write(b"*setdb172v1\r\n") # first cmd to send, set firmware ver for used sensor
s.write(b"*Zoff\r\n") # enable the control of the device by the mc

if _LOWPOWER_:
	regmwrite(CFG_REG_A_M, "10") # low power mode, 10hz data rate continuous mode
else:
	regmwrite(CFG_REG_A_M, "0C") # normal mode, 100hz data rate continuous mode

regmwrite(CFG_REG_B_M, "03") # offset cancellation and lpf
regmwrite(CFG_REG_C_M, "00") # no interrupts

if _LOWPOWER_:
	regawrite(CTRL_REG1_A, "2F") # 10hz low power mode x,y,z enabled
	regawrite(CTRL_REG4_A, "81") # enable bdu and 3-wire spi
	regawrite(TEMP_CFG_REG_A, "C0") # enable temp sensor (only works with bdu set)
else:
	regawrite(CTRL_REG1_A, "57") # 10hz hires mode x,y,z enabled
	regawrite(CTRL_REG4_A, "09") # disable bdu, high resolution and 3-wire spi
	regawrite(TEMP_CFG_REG_A, "C0") # disable temp sensor

if _LOWPOWER_:
	res = 8
else:
	res = 12

if _CALLIBRATE_:
	# https://github.com/kriswiner/MPU-6050/wiki/Simple-and-Effective-Magnetometer-Calibration
	input("start callibration")

	mxmin = mxmax = getmcoord("x")
	mymin = mymax = getmcoord("y")
	mzmin = mzmax = getmcoord("z")

	for n in range(1,128):
		mx = getmcoord("x")
		my = getmcoord("y")
		mz = getmcoord("z")

		if mx > mxmax:
			mxmax = mx
		elif mx < mxmin:
			mxmin = mx

		if my > mymax:
			mymax = my
		elif my < mymin:
			mymin = my

		if mz > mzmax:
			mzmax = mz
		elif mz < mzmin:
			mzmin = mz

		time.sleep(0.1)

	biasx = math.floor((mxmax+mxmin)/2)
	biasy = math.floor((mymax+mymin)/2)
	biasz = math.floor((mzmax/mzmin)/2)

	print("bias x: {:d} y: {:d} z: {:d}".format(biasx,biasy,biasz))

	ox = "{:04X}".format(int2twocplm(biasx))
	oy = "{:04X}".format(int2twocplm(biasy))
	oz = "{:04X}".format(int2twocplm(biasz))

	print("offsets x: "+ox+" y: "+oy+" z: "+oz)

	regmwrite("OFFSET_X_REG_L_M",ox[2:4])
	regmwrite("OFFSET_X_REG_H_M",ox[0:2])
	regmwrite("OFFSET_Y_REG_L_M",oy[2:4])
	regmwrite("OFFSET_Y_REG_H_M",oy[0:2])
	regmwrite("OFFSET_Z_REG_L_M",oz[2:4])
	regmwrite("OFFSET_Z_REG_H_M",oz[0:2])

	input("callibration done")


while _RUN_ == True:
	# register polling, we ignore DRDY
	mx = getmcoord("x")#*1.5 # 1.5: magnetic sensitivity
	my = getmcoord("y")#*1.5
	mz = getmcoord("z")#*1.5

	ax = getacoord("x")
	ay = getacoord("y")
	az = getacoord("z")

	temp = gettemp()

	gx = ax#*2/2**(res-1)
	gy = ay#*2/2**(res-1)
	gz = az#*2/2**(res-1)

	roll = math.atan2(gy, gz) # phi
	pitch = math.atan2(-gx, gy*math.sin(roll)+gz*math.cos(roll)) # theta

	x = mx*math.cos(pitch)+(my*math.sin(roll)+mz*math.cos(roll))*math.sin(pitch)
	y = mz*math.sin(roll)-my*math.cos(roll)

	heading = math.degrees(math.atan2(y,x))
	if heading<0:
		heading += 360


	print("mag x: {:.2f} y: {:.2f} z: {:.2f}".format(mx,my,mz))
	print("acc x: {:d} y: {:d} z: {:d}".format(ax,ay,az))
	print("heading: {:.2f} deg".format(heading))
	print("temp: {:d} degC".format(temp))
	time.sleep(0.3)

s.write(b"*Zon")
