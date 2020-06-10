# Lecture 3 : Conversion from Ismrmrd to Nifti format (3/3)

Title : Conversion from Ismrmrd to Nifti data

Schedule : June 11, 2020 | 15:40-17:00 

Speaker: Manon Desclides

## Summary

 - [Foreword](#foreword)
 - [A brief description of the Nifti format](#a-brief-description-of-the-nifti-format)
    - [Header](#header)
    - [Orientation](#orientation)
 - [My Nifti Convertor Gadget](#my-nifti-convertor-gadget)
    - [Getting the conversion librairy](#getting-the-conversion-librairy)
    - [Writting the conversion gadget](#writting-the-conversion-gadget)
    - [Writing the XML](#writting-the-xml)
    - [Compilation](#compilation)
    - [Reconstruction, conversion and visualisation](#reconstruction-conversion-and-visualisation)
 - [Conclusion](#conclusion)


## Foreword

This exercise will allow you to create a Python convertor gadget, to get your output data in Nifti-1 format after having reconstructed FLASH data. The convertor called in the gadget is a beta version. The version of this converter only deals with with Sagittal, Coronal and Transversal views, for all read directions and rotations (positives and negatives but not for all cases) in **HFS**. If you got any issues with the conversion, don't be surprised and tell us at the repository : https://github.com/aTrotier/ismrmrd_to_nifti/issues. Thanks !

## A brief description of the Nifti-1 format

NIfTI-1 is adapted from the widely used ANALYZE™ 7.5 file format. The hope is that older non-NIfTI-aware software that uses the ANALYZE 7.5 format will still be compatible with NIfTI-1. NIfTI-1 uses the "empty space" in the ANALYZE 7.5 header to add several new features. These innovations are summarized in the OHBM poster [PDF], and include :

* Affine coordinate definitions relating voxel index (i,j,k) to spatial location (x,y,z);
* Codes to indicate spatio-temporal slice ordering for FMRI;
* "Complete" set of 8-128 bit data types;
* Standardized way to store vector-valued datasets over 1-4 dimensional domains;
* Codes to indicate data "meaning";
* A standardized way to add "extension" data to the header;
* Dual file (.hdr & .img) or single file (.nii) storage;
and many more. The goal is to foster interoperability at the file-exchange level between FMRI data analysis software packages. The authors of AFNI, BrainVoyager, FSL, and SPM have all committed to support this format for both input and output.

(from [1](#Sources))

### Header

In order to keep compatibility with the analyze format, the size of the nifti header was maintained at 348 bytes as in the old format. Some fields were reused, some were preserved, but ignored, and some were entirely overwritten. More details on each field are provided at [2](#Sources). 

We worked with the NiBabel librairy and you can find the documentation at [3](#Sources).

### Orientation

To better understand how the data conversion works, you can find a great documentation about Nifti orientation at [4](#Sources) 





## My Nifti Convertor Gadget

### Getting the conversion librairy

First of all, you need to download the conversion librairy at : https://github.com/aTrotier/ismrmrd_to_nifti at the directory of your choice, for example in your path/to/GT_Lecture3/external_librairy.

This librairy will be called in the conversion gadget, so , copy the directory of you ismrmrd_to_nifti folder.


### Writting the conversion gadget

Create the file `nifti_python.py`  then copy the following function. You can call it `IsmrmrdToNiftiGadget`.

```
import numpy as np
import gadgetron
import ismrmrd
import sys
sys.path.append('/path/to/ismrmrd_to_nifti')
from python_version import extract_ismrmrd_parameters_from_headers as param, flip_image as fi, set_nii_hdr as tools
import nibabel as nib
import os

def IsmrmrdToNiftiGadget(connection):
    
    counter = 0
    #get header info
    hdr=connection.header

    for acquisition in connection:
        
        print("IsmrmrdToNiftiGadget, image n° ", counter)

        #Need to transpose image dimensions to be compatible with ismrmrd_to_nifti conversion code
        image = acquisition.data.transpose(3,2,1,0)

        #get the image shape
        ndim = image.shape 

        #print the image shape
        print("IsmrmrdToNifti, receiving image array of shape ", ndim)
        #print the image header
        print("IsmrmrdToNifti, receiving image head :", acquisition)
       
        ## Create parameters for set_nii_hdr et xform_mat
        h = param.extract_ismrmrd_parameters_from_headers(acquisition, hdr)
        print("IsmrmrdToNifti, computed Nifti parameters : ", h)

        ## Get crop image, flip and rotationate to match with true Nifti image
        img = image[:,:,:,0].transpose(1, 0, 2)

        ## Create nii struct based on img
        nii_empty = nib.Nifti1Image(img, np.eye(4))
        pf = {"lefthand":0}
        h2 =[]
        h2.append(h)

        ## Compute nifti parameters
        [nii_filled, h3] = tools.set_nii_hdr(nii_empty, h2, pf)

        ## Save image in nifti format
       
        output_path = os.path.join('/path/where/you/want/nifti/file/saved/',hdr.measurementInformation.measurementID + '_' + hdr.measurementInformation.protocolName+'_nifti_gadgetron.nii.gz')
        nib.save(nii_filled, output_path)
        print("Nifti well saved")
      
        #send data to next gadget
        counter += 1
        connection.send(acquisition)
              
```

This gadget gets data of the previous gadget and print the image shape and its header. Then, it creates a structure containing data parameters needed for the conversion. After that, a Nifti1 blank image is created with NiBabel librairy wich is send with the parameters structure to the *set_nii_hdr* function to compute final Nifti image.

Don't forget to change *'/path/to/ismrmrd_to_nifti'* (line 5) and */path/where/you/want/nifti/file/saved/'*  (line 46) with your own paths.

### Writing the XML

We will now create a new xml file named `python_image_array_recon_nifti_epi.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <version>2</version>

    <readers>
        <reader>
            <dll>gadgetron_core_readers</dll>
            <classname>AcquisitionReader</classname>
        </reader>
        <reader>
            <dll>gadgetron_core_readers</dll>
            <classname>WaveformReader</classname>
        </reader>
    </readers>

    <writers>
        <writer>
            <dll>gadgetron_core_writers</dll>
            <classname>ImageWriter</classname>
        </writer>
    </writers>

    <stream> 

        <gadget>
            <dll>gadgetron_mricore</dll>
            <classname>NoiseAdjustGadget</classname>
        </gadget>
        
          <!-- EPI correction -->
      <gadget>
          <name>ReconX</name>
          <dll>gadgetron_epi</dll>
          <classname>EPIReconXGadget</classname>
      </gadget>

      <gadget>
          <name>EPICorr</name>
          <dll>gadgetron_epi</dll>
          <classname>EPICorrGadget</classname>
      </gadget>

      <gadget>
          <name>FFTX</name>
          <dll>gadgetron_epi</dll>
          <classname>FFTXGadget</classname>
      </gadget>

      <gadget>
          <name>OneEncodingSpace</name>
          <dll>gadgetron_epi</dll>
          <classname>OneEncodingGadget</classname>
        </gadget>


    <gadget>
        <name>AccTrig</name>
        <dll>gadgetron_mricore</dll>
        <classname>AcquisitionAccumulateTriggerGadget</classname>
        <property>
            <name>trigger_dimension</name>
            <value>repetition</value>
        </property>
        <property>
          <name>sorting_dimension</name>
          <value>slice</value>
        </property>
    </gadget>

    <gadget>
        <name>Buff</name>
        <dll>gadgetron_mricore</dll>
        <classname>BucketToBufferGadget</classname>
        <property>
            <name>N_dimension</name>
            <value></value>
        </property>
        <property>
          <name>S_dimension</name>
          <value></value>
        </property>
        <property>
          <name>split_slices</name>
          <value>true</value>
        </property>
    </gadget>


    <gadget>
          <name>SimpleRecon</name>
          <dll>gadgetron_mricore</dll>
          <classname>SimpleReconGadget</classname>
    </gadget>

    <gadget>
          <name>ImageArraySplit</name>
          <dll>gadgetron_mricore</dll>
          <classname>ImageArraySplitGadget</classname>
    </gadget>

    <external>
            <execute name="nifti_python" target="IsmrmrdToNiftiGadget" type="python"/>
            <configuration/>
    </external> 

  </stream>

</configuration>
```

### Compilation 

Nothing to do.

### Reconstruction, conversion and visualisation

To run the reconstruction chain, you'll need to run Gadgetron, and the Gadgetron ISMRMRD client in two different terminal located in the same folder.

Start Gadgetron:
```bash
$ gadgetron
```

Run the ISMRMRD client: 
```bash 
$ gadgetron_ismrmrd_client -f Data/meas_MID00033_FID13782_gre3D_2_2_tranversal_A2P.h5.h5  -C python_image_array_recon_nifti_epi.xml
```

You will see from the Gadgetron ISMRMRD client side :

``` bash
Gadgetron ISMRMRD client
  -- host            :      localhost
  -- port            :      9002
  -- hdf5 file  in   :      ../data/meas_MID00033_FID13782_gre3D_2_2_tranversal_A2P.h5
  -- hdf5 group in   :      /dataset
  -- conf            :      default.xml
  -- loop            :      1
  -- hdf5 file out   :      out.h5
  -- hdf5 group out  :      2020-06-10 12:17:45
This measurement has dependent measurements
  SenMap : 42563_35189578_35189583_24
  Noise : 42563_35189578_35189583_24
Querying the Gadgetron instance for the dependent measurement: 42563_35189578_35189583_24
WARNING: Dependent noise measurement not found on Gadgetron server. Was the noise data processed?
```


You will see from the Gadgetron server side (example for the image 71):

```bash
-------------------------------------------------------------------------
IIsmrmrdToNifti, process ... 
(1, 52, 66, 96)
IsmrmrdToNifti, receiving image array of shape  (96, 66, 52, 1)
IsmrmrdToNifti, receiving image head : Header:
 version: 1
data_type: 7
flags: 0
measurement_uid: 0
matrix_size: 96, 66, 52
field_of_view: 215.0, 147.81199645996094, 114.4000015258789
channels: 1
position: -61.85542678833008, 52.9705924987793, -25.214649200439453
read_dir: -0.9999999403953552, 5.960464477539063e-08, 0.0
phase_dir: 5.960464477539063e-08, 0.9999999403953552, 0.0
slice_dir: 0.0, 0.0, 1.0
patient_table_position: 0.0, 0.0, -1621005.0
average: 0
slice: 0
contrast: 0
phase: 0
repetition: 0
set: 0
acquisition_time_stamp: 21877162
physiology_time_stamp: 0, 0, 0
image_type: 0
image_index: 1
image_series_index: 0
user_int: 0, 0, 0, 0, 0, 0, 0, 0
user_float: 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0
attribute_string_len: 0


Attribute string:
 
Data:
 [[[[1.12745820e-05+0.j 1.28176371e-05+0.j 1.12684647e-05+0.j ...
    1.46980137e-05+0.j 1.06317939e-05+0.j 1.20189488e-05+0.j]
   [1.20199693e-05+0.j 1.55809612e-05+0.j 1.14457043e-05+0.j ...
    1.35915807e-05+0.j 9.04514673e-06+0.j 1.18929065e-05+0.j]
   [1.25115475e-05+0.j 1.22919519e-05+0.j 1.23482014e-05+0.j ...
    9.70610745e-06+0.j 1.01434298e-05+0.j 1.05357922e-05+0.j]
   ...  

   ...
   [1.13031456e-05+0.j 1.27430931e-05+0.j 1.12881198e-05+0.j ...
    1.15715839e-05+0.j 1.20293553e-05+0.j 9.76563933e-06+0.j]
   [9.84120379e-06+0.j 1.26899913e-05+0.j 1.11872851e-05+0.j ...
    1.15753101e-05+0.j 1.25896358e-05+0.j 1.05092031e-05+0.j]
   [1.08456788e-05+0.j 1.54542013e-05+0.j 1.30059934e-05+0.j ...
    1.54069003e-05+0.j 1.39716667e-05+0.j 1.44729429e-05+0.j]]]]

IsmrmrdToNifti, computed Nifti parameters :  {'NumberOfTemporalPositions': 1, 'MRAcquisitionType': '3D', 'ImageOrientationPatient': array([-5.9604645e-08, -9.9999994e-01, -0.0000000e+00,  9.9999994e-01,
       -5.9604645e-08, -0.0000000e+00], dtype=float32), 'SpacingBetweenSlices': 2.2000000384615386, 'SliceThickness': 2.2000000384615386, 'PixelSpacing': array([[2.23958333],
       [2.2395757 ]]), 'ImagePositionPatient': array([[-169.35541598],
       [ 126.8765925 ],
       [  31.9853518 ]]), 'LastFile': {'ImagePositionPatient': array([[-169.35541598],
       [ 126.8765925 ],
       [ -82.4146502 ]])}, 'Manufacturer': 'SIEMENS', 'Columns': 66, 'NiftiName': 'gre3D_2.2_tranversal_A2P', 'InPlanePhaseEncodingDirection': 'COL'}
/tmp/gadgetron/42563_35189578_35189583_33_gre3D_2.2_tranversal_A2P_nifti_manon_gadgetron.nii.gz
Nifti well saved
-------------------------------------------------------------------------
06-10 12:17:47.403 DEBUG [ext. 794 nifti_python.IsmrmrdToNiftiGadget] Connection closed normally.
06-10 12:17:47.404 DEBUG [Gadget.h:130] Shutting down Gadget ()
06-10 12:17:47.404 DEBUG [Gadget.h:130] Shutting down Gadget ()
06-10 12:17:47.404 DEBUG [Gadget.h:130] Shutting down Gadget ()
06-10 12:17:47.461 INFO [Core.cpp:76] Connection state: [FINISHED]


```

## Visualisation

Use itksnap or 3DSlicer to load the nifti file : /tmp/gadgetron/42563_35189578_35189583_33_gre3D_2.2_tranversal_A2P_nifti_manon_gadgetron.nii.gz

## Conclusion


This conclude the conversion to nifti from ismrmrd format after reconstruction. You can adapt this gadget chain to reconstruct other type of data, by replacing the reconstruction part by the one you want !

If you have any issues or ideas to enhance this gadget or the ismrmrd_to_nifti librairy, don't hesitate to tell us at https://groups.google.com/forum/#!msg/gadgetron/_ob_fKaSfqo/qV3z3znbGwAJ or  https://github.com/aTrotier/ismrmrd_to_nifti/issues. Thanks ! 
 


## Sources
1. [dfwg. "NIfTI: — Neuroimaging Informatics Technology Initiative". nifti.nimh.nih.gov. Retrieved 2019-08-03.](https://nifti.nimh.nih.gov/nifti-1/)
2. [Brainder. "The NIFTI file format". brainder.org/2012/09/23/the-nifti-file-format/. Posted 2012-23-09](https://brainder.org/2012/09/23/the-nifti-file-format/)
3. [Chris Markiewicz. "Working with NIfTI images". nipy.org. ](https://nipy.org/nibabel/nifti_images.html)
4. [Chris Markiewicz. "Coordinate systems and affines". nipy.org. ](https://nipy.org/nibabel/coordinate_systems.html)