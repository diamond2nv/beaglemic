# BeagleMic 8-channel PDM Audio Capture

# Introduction
Ever wanted to record audio from 8 PDM microphones simultanously? Now you can with a BeagleMic running on a [PocketBeagle](https://beagleboard.org/pocket) or [BeagleBone AI](https://bbb.io/ai)

Yes, you could opt for the much simpler I2S microphones. But then you won't have fun writing assembly to capture and process sixteen digital signals at more than 2MHz sample rate.

Current firmware supports:

| Feature                    | Support           |
|----------------------------|-------------------|
| PDM Bit Clock              | 3,448 MHz         |
| PCM Output Bits Per Sample | 24 bps            |
| PCM Output Sample Rate     | 26939 Samples/sec |

# Hardware
The schematic is simple. PDM microphones' digital outputs are connected directly to the PRU input pins. The PRU also drives the bit clock. I have tested on PocketBeagle and BeagleBone AI.

| PocketBeagle | BBAI  | PRU Pin | Type  | Signal               |
|--------------|-------|---------|-------|----------------------|
| P2.24        | P9.11 | R30_14  | Output| PDM Bit Clock        |
| P1.36        | P8.44 | R31_0   | Input | MIC0                 |
| P1.33        | P8.41 | R31_1   | Input | MIC1                 |
| P2.32        | P8.42 | R31_2   | Input | MIC2                 |
| P2.30        | P8.39 | R31_3   | Input | MIC3                 |
| P1.31        | P8.40 | R31_4   | Input | MIC4                 |
| P2.34        | P8.37 | R31_5   | Input | MIC5                 |
| P2.28        | P8.38 | R31_6   | Input | MIC6                 |
| P1.29        | P8.36 | R31_7   | Input | MIC7                 |


Optionally, high-level software may visualize detected audio direction using a stripe of 16 LEDs hooked to two 74HC595 shift registers:

| PocketBeagle | PB GPIO  | BBAI  | BBAI GPIO | Signal |
|--------------|----------|-------|-----------|--------|
| P2.25        | gpio2_9  | P9.30 | gpio5_12  | DS     |
| P2.29        | gpio1_7  | P9.28 | gpio4_17  | SHCP   |
| P2.31        | gpio1_19 | P9.23 | gpio7_11  | STCP   |


Microphone breakout board and a PocketBeagle Cape are provided in KiCad format.

Unfortunately the breakout board is essential for a home DIY user like me, since all PDM microphones I could find are BGA. There are numerous youtube guides how to solder BGAs at home using a skillet or a toaster oven.

# Software
PRU0 takes care of driving the PDM bit clock and capturing the microphone bit data. It then runs a CIC filter to convert PDM to PCM, and feeds PCM data to PRU1.

PRU1 is retransmitting PCM data from PRU0 down to the ARM host. RPMSG is used for control commands. Shared DDRAM buffers are used for audio data.

Host audio driver presents a standard ALSA audio card, so that arecord and other standard tools can be readily used.

# Running The Example on PocketBeagle

    # Build kernel module
    sudo apt update
    sudo apt install build-essential linux-headers-`uname -r`
    cd driver
    make
    # Install kernel module
    sudo cp beaglemic.ko /lib/modules/`uname -r`/kernel/drivers/rpmsg/
    sudo depmod -a

    # Load the driver
    sudo modprobe beaglemic

    # Build PRU firmware
    cd pru
    make
    # Start the new PRU firmware
    sudo ./start.sh

    # Record audio
    arecord  -r26939  -c8 -f S32_LE -t wav out.wav
    # Hit Ctrl+C to stop.

# Running The Example on BeagleBone AI

    # Prebuilt DTBO for BB AI is provided in this repo. For corresponding
    # source change, see driver/0001-Initial-BeagleMic-DTS-for-BBAI.patch
    sudo cp driver/am5729-beagleboneai-beaglemic.dtb /boot/dtbs/`uname -r`/
    sudo sed -i -e 's@#dtb=@dtb=am5729-beagleboneai-beaglemic.dtb@g' /boot/uEnv.txt
    sync
    sudo reboot

    # Build kernel module
    sudo apt update
    sudo apt install build-essential linux-headers-`uname -r`
    cd driver
    make
    # Install kernel module
    sudo cp beaglemic.ko /lib/modules/`uname -r`/kernel/drivers/rpmsg/
    sudo depmod -a

    # Load the driver
    sudo modprobe beaglemic

    # Build PRU firmware
    cd pru
    make
    # Start the new PRU firmware
    sudo ./start-bbai.sh

    # Record audio. Use second audio card, since first one is
    # the onboard HDMI.
    arecord  -D hw:1,0 -r26939  -c8 -f S32_LE -t wav out.wav
    # Hit Ctrl+C to stop.


# Folder Structure

 * beaglemic-cape - Universal cape for BeagleBone AI, PocketBeagle and iCE40HX8K-EVB. **Work in progress.**
 * driver - Host ALSA audio driver.
 * inmp621-breakout - INMP621 Microphone breakout. Designed to be optionally used with beaglemic-cape. **Work in progress.**
 * libs - Library code. Currently contains code to drive LED ring for beaglemic-cape.
 * pru - PRU firmware.
 * deprecated/pocket-cape - PocketBeagle cape to be used with deprecated/spm0423hd4h-breakout.
 * deprecated/spm0423hd4h-breakout - SPM0432HD4H microphone breakout. Designed to be used with pocket-cape.

# Further Work
A few ideas to improve the design:

 * Can the PRU subsystems in BBAI process 8 channels each, to combine for 16 microphone capture?
 * Move comb filters to PRU1, and try to add more integrators in PRU0.
 * Clean-up the cape PCB.
   * If possible, leave headers for Class-D output from spare PRUs.

# References
 * [CIC Filter Introduction](https://dspguru.com/dsp/tutorials/cic-filter-introduction/)
 * [Another CIC Filter Article](http://www.tsdconseil.fr/log/scriptscilab/cic/cic-en.pdf)
 * [Series of PDM articles and software implementations](https://curiouser.cheshireeng.com/category/projects/pdm-microphone-toys/)
 * [Another CIC Filter Article](https://www.embedded.com/design/configurable-systems/4006446/Understanding-cascaded-integrator-comb-filters)
 * [CIC Filter Introduction](http://home.mit.bme.hu/~kollar/papers/cic.pdf)
 * [Datasheet for the PDM microphones I've used](http://media.digikey.com/PDF/Data Sheets/Knowles Acoustics PDFs/SPM0423HD4H-WB.pdf)
 * [Inspiration for high-bandwidth data acquisition](https://github.com/ZeekHuge/BeagleScope)

# Other Resources
Below are various other CIC implementations and PDM microphone boards I've stumbled upon.
 * https://github.com/introlab/16SoundsUSB
 * https://github.com/introlab/odas
 * https://respeaker.io/make_a_smart_speaker/
 * https://github.com/Scrashdown/PRU-Audio-Processing
 * https://github.com/fakufaku/kurodako
