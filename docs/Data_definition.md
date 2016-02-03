# FOXSI Data Description

* version 4, 2015 Jan 06 update: Revamped data and software to work with FOXSI-2 data.  LG

## Data location

All data, from the raw recorded files through Level 2 data, is located in data_2012 and data_2014 folders at:

   ftp://apollo.ssl.berkeley.edu/pub/foxsi

For analysis purposes, you only need the Level 1 and Level 2 processed data files; you don’t need the raw data.  Calibration data files are also needed; these are described in the FOXSI software documentation.

## Recorded data files

Recorded flight data is in two formats:

1. File recorded on FOXSI GSE computer: a binary file identical in format to the calibration files with the formatter interface.  Each data frame is 256 words starting with the second sync word 0xF628.

    * FOXSI-1 Filename:  data_121102_114631.dat
    * FOXSI-2 Filename:  data_141211_115911.dat

2.	File recorded by WSMR ground station: the official data record.  This binary file contains our data frames with 3 additional words inserted between the second sync word (F628) and the start of our housekeeping data.  The three extra words contain time and sync lock info from the ground station.  This file was recorded upstream of our GSE computer and thus has fewer opportunities for sync problems (though note there seems to be a 36 time lag for FOXSI-2, cause unknown).

    * FOXSI-1 Filename:  36.255_TM2_Flight_2012-11-02.log
    * FOXSI-2 Filename:  36_295_Krucker_FLIGHT_HOT_TM2.log

**Note that some dropouts and sync problems are present in both files, on the order of 0.1% of the frames.  However, most of these dropouts are preflight or during burnouts.**

## IDL data structures

The data volume is sufficiently small that data can be stored as IDL structures in IDL save files, as opposed to fits files.

### Level 0 data

The FOXSI level 0 data structure captures all the unprocessed, raw data, organized with each element representing one triggered event in one detector.  Thus not all data frames have events, and there can be more than one event (in multiple detectors) per frame.  The trigger can be caused by a photon interaction or by noise; in this documentation we’ll call both of these hits.  For convenience, the hit ASIC, strip, and ADC values are specifically called out for the two sides– these are determined by identifying which ASIC or strip had the maximum value on each side (at this stage, the calculation is done without subtracting common mode values).  However all the recorded detector data, including all transmitted strip data and channel masks, are included in the structure.  Together with the housekeeping structure, the level 0 data carries all of the information from the recorded data file, and the raw data file should no longer be needed.  Some data frames are flagged as bad packets based on the common mode values, but no data is thrown away at this point, unless all words for that detector are zero in the frame (i.e. no trigger).  Events that occurred after the rocket launched and HV ramp neared 200V have an “inflight” flag set.

**Note that the time recorded by WSMR is in UT and has microsecond precision BUT time delays of transmission from 100-300km limit this to ~ms precision unless we do additional refinement.  Also, this time is the frame time recorded at the ground station, not the detector trigger time onboard the rocket payload, adding additional uncertainty of 1-2 ms.  More precise timing could be calculated for higher level data if necessary, but precision of a few ms is expected to be sufficient for FOXSI data analysis.  WSMR times recorded in the data structure will only have significant digits to the millisecond place.
***For FOXSI-2 data only, there is a 36-second delay in the ground station recorded time.  This delay is of unknown origin.

* Level 0 data file:  foxsi_level0_data.sav

Structures in .sav file:
* data_lvl0_D0
* data_lvl0_D1
* data_lvl0_D2
* data_lvl0_D3
* data_lvl0_D4
* data_lvl0_D5
* data_lvl0_D6

Tags:
*	Frame number
*	Time (in seconds-of-day, from WSMR.  See earlier notes on timing.)
*	Frame time
*	Detector number
*	Trigger time
*	Hit ASIC # [n-side, p-side]
*	Hit strip # [n-side, p-side]
*	Hit ADC value [n-side, p-side]
*	Strip numbers with recorded data (all ASICs), 4x3 array
*	3-strip value (all ASICs), 4x3 array
*	Common mode values (all ASICs) 4x1 array
*	Channel masks (all ASICs) 4x1 array with each mask encoded in a 64-bit long
*	Detector bias voltage (“HV”) in raw form (including status bit)
*	Detector temperature, if we have it, from nearest applicable frame
*	In flight: this flag is set if the event occurred while HV>190V (FOXSI-1).
*	Altitude, in meters.  Altitude values are repeated between 0.5 sec cadence, not interpolated.
*	Error flag: set if any common mode value is out of range.  However, no data is thrown away at this point!

### Level 1 data

The intention of level 1 data is to include higher-level information but to exclude any intricate processing steps that could be expected to change later as we make refinements (i.e. detector response, payload pointing, etc).  ASIC and strip numbers are replaced with 2D position information, given in both detector coordinates (pixels) and in payload coordinates (arcseconds), using a coordinate system similar to that of the GSE, but with the vertical reflection fixed.  The “detector” coordinate system is arbitrary and won’t necessarily match the “zoom” window on the GSE.  For this coordinate system the p- coordinate is given first.

Coarse payload coordinates means that the design-specified geometric rotation of each detector is taken into account.  A position in fine payload coordinates is subject to coregistration and so will be part of the next level.

Frame and trigger times are used to calculate a livetime value (for this frame only; doesn’t include multiple frames).  Within each event, strip values are classified as (a) “hit” (highest value for that side of the detector), (b) “associated” (values came from the hit ASIC), and (c) “unrelated” (values from the other ASIC).

**IMPORTANT:  As of FOXSI-2, Level 1 data is being truncated to not include events before the HV reaches 200V.  For FOXSI-1, all events were carried through all levels of processing, so that preflight data could be used for troubleshooting and diagnostics.  This is no longer necessary, so FOXSI-2 data only includes HV=200V data for Level 1 and Level 2.  FOXSI-1 data is being retained as before for backwards compatibility.**

Level 1 data file:  **foxsi_level1_data.sav**

Structures in .sav file:
* data_lvl1_D0
* data_lvl1_D1
* data_lvl1_D2
* data_lvl1_D3
* data_lvl1_D4
* data_lvl1_D5
* data_lvl1_D6

Tags:
*	Frame number
*	Time (in seconds-of-day, from WSMR.  See earlier notes on timing.)
*	Frame time
*	Detector number
*	Trigger time
*	Livetime (for this frame only!!  Doesn’t include extra frames since last event yet.)
*	Hit ADC values [n-side, p-side], no common-mode subtraction
*	Common mode value for the hit ASIC (also use this CM for the associated strips)
*	Hit ADC values [n-side, p-side], common-mode subtracted
*	Hit 2D position in detector coordinates [p-side, n-side] [pixels]
*	Hit 2D position in coarse payload coordinates [x, y] [arcsec]
*	Associated ADC values, 3x3x2 array (includes hit!) (last dim is n- or p-side)
*	Associated positions, 3x3x2 (includes hit!) (last dim is n- or p-side)
*	Unrelated ADC values, 3x3x2 array (last dim is n- or p-side)
*	Unrelated positions, 3x3x2 (last dim is n- or p-side)
*	Unrelated common mode values [n-side, p-side]
*	Channel masks (all ASICs) 4x1 array with each mask encoded in a 64-bit long.
o	A “pixel map” is not possible because of the large memory needed to store a 128x128 array for a structure with 3^5 elements.
*	Detector bias voltage (“HV”) in volts
*	Detector temperature, if we have it, from nearest applicable frame
*	In flight: this flag is set if the event occurred when HV>190V.
*	Altitude, in meters.  Altitude values are interpolated between 0.5 sec cadence.
*	Error flag, stored bitwise.  So far 7 suspicious red flags have been identified; each of these gets a bit.  By storing the values bitwise, more than one error can be flagged.  The bits as of 2/19/2013 are:
	   * Bit 0: Any CM value > 1023
       * Bit 1: All zero ADC data from one side (either n or p)
       * Bit 2: p-side CM is 0 (can't use value for spectroscopy)
       * Bit 3: Detector voltage possibly not settled (applies for ~40 sec after HV ramp finishes and at end, when HV ramp down starts)
       * Bit 4: CM > highest ADC for that ASIC.  (This mostly duplicates bit 0 error.)
       * Bit 5: Signal location is within 3 strips from detector edge.
       * Bit 6: Livetime value out of range ([1,2000] us)

It is emphasized that an error flag doesn’t necessarily mean the event should be thrown out (e.g. we may get good values before the detector current settles, and incomplete readouts may still have useful data in them) but these values should be used with caution.  For the most conservative analysis, include only data with no flagged errors.

### Level 2 data

Level 2 conversion includes most calibration steps.  ADC values are replaced with energies (diagonal matrix elements only).  SPARCS pointing information is included in order to translate coordinates from detector/payload into heliocentric.

Level 2 data file:  **foxsi_level2_data.sav**

Structures in .sav file:
* data_lvl2_D0
* data_lvl2_D1
* data_lvl2_D2
* data_lvl2_D3
* data_lvl2_D4
* data_lvl2_D5
* data_lvl2_D6

Tags:
*	Frame number
*	Time (in seconds-of-day, from WSMR.  See earlier note on timing.)
*	Frame time
*	Detector number
*	Trigger time
*	Livetime (for this frame only!!  Doesn’t include extra frames since last event yet.)
*	Hit keV values [n-side, p-side], common-mode subtracted
*	**included as of 2014 Feb. 16**  Hit 2D position in detector coordinates [p-side, n-side] [pixels]
*	Hit 2D position in payload coordinates [x, y] [arcsec]
*	Hit 2D position in heliocentric coordinates [x, y] [arcsec]
*	Associated keV values, 3x3x2 array (includes hit!) (last dim is n- or p-side)
*	Associated positions, 3x3x2 (includes hit!) (last dim is n- or p-side)
*	Detector bias voltage (“HV”) in volts
*	Detector temperature, if we have it, from nearest applicable frame
*	In flight: this flag is set if the event occurred when HV>190V
*	Altitude, in meters.  Altitude values are interpolated between 0.5 sec cadence.
*	SPARCS pitch: gives the payload coord origin on the Sun, horizontal component
*	SPARCS yaw: gives the reverse of the payload coord origin on the Sun, vertical component
*	Error flag, stored bitwise.  So far 7 suspicious red flags have been identified; each of these gets a bit.  By storing the values bitwise, more than one error can be flagged.  The bits as of 2/19/2013 are:
    * Bit 0: Any CM value > 1023
    * Bit 1: All zero ADC data from one side (either n or p)
    * Bit 2: p-side CM is 0 (can't use value for spectroscopy)
    * Bit 3: Detector voltage possibly not settled (applies for ~40 sec after HV ramp finishes and at end, when HV ramp down starts)
    * Bit 4: CM > highest ADC for that ASIC.  (This mostly duplicates bit 0 error.)
    * Bit 5: Signal location is within 3 strips from detector edge.
    * Bit 6: Livetime value out of range ([1,2000] us)

It is emphasized that an error flag doesn’t necessarily mean the event should be thrown out (e.g. we may get good values before the detector current settles, and incomplete readouts may still have useful data in them) but these values should be used with caution.  For the most conservative analysis, include only data with no flagged errors.

## Housekeeping Data

Housekeeping data is stored separately from the event data because it arrives regularly either every frame (voltages) or every fourth frame (temperatures), instead of sporadically like the detector events do.  Each entry in the housekeeping data structure corresponds to one data frame.  Housekeeping values that show up every fourth frame are allowed to “leak” into neighboring frames to provide a value for every frame.  This is also done for the altitude data (cadence 0.5 sec) to match the formatter frame cadence (0.002 sec).

## Level 0

All data are raw values, meaning that thermistor A/D values have not been decoded into real temperatures, voltages, etc.

Level 0 data file:  **foxsi_level0_hskp_data.sav**

Structures in .sav file:  **hskp_data**

Tags:
1. Frame counter
2. Time (in seconds-of-day, from WSMR.  See earlier note on timing.)
3. Formatter frame time
4. High voltage value (verbatim from the frame including status bits)
5-8) All voltages (raw values)
9-20) All thermistors on electronics package (raw values)
21-24)  Additional formatter housekeeping words (command count, command1, command2, formatter status)
25-31)  Status word for each detector – (Note: probably no meaningful data here.)
32)  In-flight flag: ‘1’ for frames after launch
33)  Error flag: ‘1’ if obvious error is detected (in this case if frame counter does not increment by 1)
34.  Altitude, in meters.  Altitude values are repeated between 0.5 sec cadence, not interpolated.

## Level 1

The level 1 housekeeping data structure is identical to the level 0 structure except that all values are converted into volts and degrees.
