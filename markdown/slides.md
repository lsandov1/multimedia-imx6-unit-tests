# IPU/VPU `i.MX6` Unit Tests

Unit tests showing the most features on the `iMX.6`

## Including the package into an image

* Enable the package named `imx-test` on your Yocto image

~~~~{.python}
require recipes-core/images/core-image-minimal.bb
CORE_IMAGE_EXTRA_INSTALL += "imx-test"
~~~~

* The image `fsl-image-machine-test` has the latter package included

## Cross-compiling **NOT TESTED**

One option is to cross-compile the tarball including the unit tests

* Download 

~~~~{.bash}
wget \
    http://www.freescale.com/lgfiles/NMG/MAD/YOCTO//imx-test-3.10.17-1.0.0.tar.gz
~~~~

* Compilation

~~~~{.bash}
LDFLAGS=""
PLATFORM=IMX6Q
LINUXPATH=/home/b42214/yocto/community/daisy/fsl-community-bsp/build/tmp/sysroots/imx6qsabresd/usr/src/kernel
KBUILD_OUTPUT=/home/b42214/yocto/community/daisy/fsl-community-bsp/build/tmp/sysroots/imx6qsabresd/usr/src/kernel 
CROSS_COMPILE=arm-poky-linux-gnueabi-
V=1
INC="-I/home/b42214/yocto/community/daisy/fsl-community-bsp/build/tmp/sysroots/imx6qsabresd/usr/include \
     -I/home/b42214/yocto/community/daisy/fsl-community-bsp/build/tmp/work/imx6qsabresd-poky-linux-gnueabi/imx-test/1_3.10.17-1.0.0-r0/imx-test-3.10.17-1.0.0/include \
     -I/home/b42214/yocto/community/daisy/fsl-community-bsp/build/tmp/sysroots/imx6qsabresd/usr/src/kernel/include/uapi \
     -I/home/b42214/yocto/community/daisy/fsl-community-bsp/build/tmp/sysroots/imx6qsabresd/usr/src/kernel/include 
     -I/home/b42214/yocto/community/daisy/fsl-community-bsp/build/tmp/sysroots/imx6qsabresd/usr/src/kernel/arch/arm/include \
     -I/home/b42214/yocto/community/daisy/fsl-community-bsp/build/tmp/sysroots/imx6qsabresd/usr/src/kernel/drivers/mxc/security/rng/include \
     -I/home/b42214/yocto/community/daisy/fsl-community-bsp/build/tmp/sysroots/imx6qsabresd/usr/src/kernel/drivers/mxc/security/sahara2/include"
make
~~~~

* Installation

~~~~{.bash}
ROOTFS=<where the rootfs is mounted>
sudo mkdir $ROOTFS/unit_tests
sudo cp test-utils.sh $ROOTFS/unit_tests
sudo cp platform/IMX6Q/* $ROOTFS/unit_tests/
sudo cp clocks.sh $ROOTFS/unit_tests/
~~~~

## IPU

* Image Processing Unit

* Designed to support:
    * Video and graphics processing functions
    * Interface with video and still image sensors and displays

* The IPU driver provides a kernel-level API to manipulate **logical channels**

* Logical Channel represents a complete IPU processing flow:

    Reading a YUV buffer --> post processing --> RGB buffer to memory

## IPU: Scaling

~~~~{.bash}
/unit_tests/mxc_ipudev_test.out -h
~~~~

* Down-scale an image

~~~~{.bash}
/unit_tests/mxc_ipudev_test.out -c 1 \
                      -i 1024,768,RGBP,0,0,0,0,0,0 \
                      -O 176,144,RGBP,0,0,0,0,0 \
                      -s 0 \
                      -f smallsize.rgbp \
                      wall-1024x768-565.rgb
~~~~

## IPU: Color-Space Conversion

* Down-scale and change the color space convert (RGB to YUV)

~~~~{.bash}
/unit_tests/mxc_ipudev_test.out -c 1 \
                      -i 1024,768,RGBP,0,0,0,0,0,0 \
                      -O 176,144,UYVY,0,0,0,0,0 \
                      -s 0 \
                      -f smallsize.yuv \
                      wall-1024x768-565.rgb
~~~~

* [Color Space Formats](http://www.fourcc.org/)

## IPU: Camera Video Capture

* Make sure you have the related camera modules (`ov5642_camera` or `ov5640_camera`) and the
  `mxc_v4l2_capture` module, the latter needed to support the [*Video4Linux*](http://en.wikipedia.org/wiki/Video4Linux) API

~~~~{.bash}
root@imx6qsabresd:/unit_tests# lsmod
Module                  Size  Used by
ov5642_camera          75119  0 
mxc_v4l2_capture       22274  1 
ov5640_camera          17959  0 
ipu_bg_overlay_sdc      4433  1 mxc_v4l2_capture
ov5640_camera_mipi     20774  0 
ipu_still               1663  1 mxc_v4l2_capture
ipu_prp_enc             4645  1 mxc_v4l2_capture
ipu_csi_enc             3210  1 mxc_v4l2_capture
ipu_fg_overlay_sdc      5449  1 mxc_v4l2_capture
evbug                   1476  0
~~~~~~

* Create a RAW video, VGA, 5 seconds @ 30fps, no rotation. The format will be `YUV420`.

~~~~{.bash}
W=640
H=480
FPS=30
SEC=5
OUTFOLDER=/var/volatile
OUTYUV=file.yuv
OUT=${OUTFOLDER}/${OUTYUV}
TOTALFRAMES=`expr $SEC \* $FPS`
~~~~~

~~~~{.bash}
/unit_tests/mxc_v4l2_capture.out \
-iw $W -ih $H -ow $W -oh $H -r 0 -c $TOTALFRAMES -fr $FPS $OUT
~~~~

* Show the RAW video (just to make sure it was recorded correctly)

~~~~{.bash}
/unit_tests/mxc_v4l2_output.out -iw $W -ih $H $OUT
~~~~

## IPU: Camera Image Capture

* A still image at the resolution of 640x480 will be saved in the file `still.yuv` 
  in the format `YUV422` planar.

~~~~{.bash}
/unit_tests/mxc_v4l2_still.out -w 640 -h 480 -f YUV422P
~~~~

The image can be viewed with:

~~~~{.bash}
/unit_tests/mxc_v4l2_output.out \
-iw 640 -ih 480 -ow 176 -oh 144 -l 100 -f 422P still.yuv
~~~~


## IPU: Camera Overlay

* Direct preview the camera to the foreground, and set the frame rate to 30 fps, 
the window of interest is 640 X 480 with a starting offset of (0,0), 
the preview size is 160 X 160 with a starting offset of (20,20). 

* For `HDMI` output, so may need to add the 
  `video=mxcfb0:dev=hdmi,1920x1080M@60,if=RGB24` argument to the kernel
  command line

~~~~{.bash}
/unit_tests/mxc_v4l2_overlay.out -help
/unit_tests/mxc_v4l2_overlay.out \
-iw 640 -ih 480 -it 0 -il 0 -ow 160 -oh 160 -ot 20 -ol 20 -r 0 -t 50 -do 0 -fg -fr 30
~~~~


## VPU: Video Encoding

~~~~{.bash}
# To test MPEG-4 encode:
/unit_tests/mxc_vpu_test.out \
-E "-i $OUT -w $W -h $H -f 0 -o $OUTFOLDER/file.mpeg4"

# To test H.263 encode:
/unit_tests/mxc_vpu_test.out \
-E "-i $OUT -w $W -h $H -f 1 -o $OUTFOLDER/file.263"

# To test H.264 encode:
/unit_tests/mxc_vpu_test.out \
-E "-i $OUT -w $W -h $H -f 2 -o $OUTFOLDER/file.264"
~~~~

* Compare files' size

~~~~{.bash}
root@imx6qsabresd:/unit_tests# ls -lah $OUTFOLDER/file*
-rwxr-xr-x    1 root     root        4.5M Oct  1 18:51 file.263
-rwxr-xr-x    1 root     root        5.6M Oct  1 18:51 file.264
-rwxr-xr-x    1 root     root        3.3M Oct  1 18:51 file.mpeg4
-rw-r--r--    1 root     root       65.9M Oct  1 17:55 file.yuv
root@imx6qsabresd:/unit_tests#
~~~~

## VPU: Video Decoding

~~~~{.bash}
# To test MPEG-4 decode:
/unit_tests/mxc_vpu_test.out \
-D "-i $OUTFOLDER/file.mpeg4 -f 0 -o $OUTFOLDER/file_mpeg4.yuv"

# To test H.263 decode:
/unit_tests/mxc_vpu_test.out \
-D "-i $OUTFOLDER/file.263 -f 1 -o $OUTFOLDER/file_263.yuv"

# To test H.264 decode:
/unit_tests/mxc_vpu_test.out \
-D "-i $OUTFOLDER/file.264 -f 2 -o $OUTFOLDER/file_264.yuv"
~~~~

* Compare checksums

~~~~{.bash}
root@imx6qsabresd:/unit_tests# md5sum /var/volatile/file*
d2a88cc4dad600acfb66f1fa4ab0c2fd  /var/volatile/file.263
31fe0bd472807178ea7ad58abf367421  /var/volatile/file.264
c53083cfcda7a7c8320b01508706c8de  /var/volatile/file.mpeg4
322f065a06e16ae57f25f8677c488cdf  /var/volatile/file.yuv
a8e2bd9837c63672429e22d25ffdaef0  /var/volatile/file_263.yuv
7f2d6eaba2e26c5497e20b1392ce7d6b  /var/volatile/file_264.yuv
4dd4cc8c694c928ae0ede542cc7cca15  /var/volatile/file_mpeg4.yuv
~~~~

## Lossy and Lossless Compression

    Lossless compression codecs preserve all of the information contained 
    within the original file. Lossy codecs, on the other hand, discard 
    some data contained in the original file during compression... 

    Lossy codecs generally let you specify a target 
    data rate, and discard enough information to hit that data rate target. 


    Compression for Great Video and Audio, 2nd Edition
    by Ben Waggoner


