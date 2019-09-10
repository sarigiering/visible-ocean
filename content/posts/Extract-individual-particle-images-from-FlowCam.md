+++
title = "Extract individual particle images from FlowCam"
date = 2019-09-09T11:20:08-02:30
draft = false
tags = ["particles", "FlowCam", "vignette", "extraction"]
categories = []
+++

FlowCam is a great device for imaging plankton samples. It takes photos of the particles in your sample and analyzes them using the provided software, VisualSpreadsheet, which carries out particle detection, classification, measurements, and statistical analyses. VisualSpreadsheet is great for most purposes, but sometimes you may want to access the individual particle images ('vignettes') for more flexible post-processing.

I here provide a code allowing you to extract the individual images. I assume that you have already analyzed several samples using FlowCam. In the near future, I will write a separate tutorial about what we found are the best settings to image marine snow.

![Example of FlowCam collage image](/img/2019_09_09_FlowCam_collage_example.png)

## Sample folder structure
After you run your sample through the FlowCam, you will have saved a sample folder containing the following files:

* cal\_image\_??????.tif
* \<*sample_name*>.ctx
* \<*sample_name*>.edg
* \<*sample_name*>.lst
* \<*sample_name*>\_??????.tif
* \<*sample_name*>\_??????\_bin.tif
* \<*sample_name*>\_run\_summary.txt

where ?????? is a six-digit number starting at 000001. The folder may contain several files of this type numbered sequentially.

## Access individual vignettes
VisualSpreadsheet saves the individual vignettes in collage files. The information about each vignette and where it is located in the collage file is provided in the metadata table called `<sample_name>.lst`. To extract individual vignettes, I use the information provided in the metadata table.

Note that the coordinates of the image are listed as you would for a matrix:

1. start at top-left corner
2. coordinates are given as [y,x], equivalent to [row,column]

The position of the vignette in the composite image is provided by
  *image_y* and *image_x*. The height and width of the vignette are provided as:
  *image_h* and *image_2*.

The code will fail if any of the collage images are missing.

## Set up your directory for extraction
Copy all sample folders that you want to have extracted in a folder; e.g. `~/<Project_name>/<raw>`. I named it "raw", but you can name it anything you want.

Run the code below. When prompted select the `<raw>` folder.

Vignettes will be saved in a folder called `~/<Project_name>/Extracted`.

This is how your directory might look like before you run the code:
```
.
...
├── <Project_name>
|   ├── <raw>
|   |   ├── <Sample_1>
|   |   |   ├── *.lst
|   |   |   ├── *_000001.tif
|   |   |   ...
|   |   ├── <Sample_2>
|   |   |   ├── *.lst
|   |   |   ├── *_000001.tif
|   |   |   ...
...
```

After extraction, you will find a new folder ("Extracted") in your project folder containing subfolders for each sample with the extracted vignettes.

```
.
...
├── <Project_name>
|   ├── ...
|   └── Extracted
|   |   ├── <Sample_1>
|   |   |   ├── *_vign000001.png
|   |   |   ├── *_vign000002.png
|   |   |   ...
|   |   ├── <Sample_2>
|   |   |   ├── *_vign000001.png
|   |   |   ├── *_vign000002.png
|   |   |   ...
...
```

## Code 
Copy-paste the code below into your Python environment (e.g. Spyder). You should not have to change anything in the code, though you may have to install some of the packages. To extract your vignettes, simply select the entire code and run everything (Ctrl+A, Ctrl+Enter).

The code will extract all vignettes in your selected `<raw>` folder, even if they have previously been extracted. So, don't forget to remove the files that you don't want to extract again or choose a different `<raw>` folder.

```py
# Python 3.7
import os, os.path
import pandas
import cv2
import tkinter as tk
from tkinter import filedialog

# prompt for input folder (<raw>)
root = tk.Tk()
root.attributes('-topmost', 1)
root.withdraw()
raw_folder_path = filedialog.askdirectory()
root.update()
os.chdir(raw_folder_path)

# make directory for extraction
extract_folder_path = os.path.join(os.path.dirname(raw_folder_path),"Extracted")
if not os.path.exists(extract_folder_path): os.mkdir(extract_folder_path)

# print folder names
print("Selected folder: " + raw_folder_path)
print("Files will be extracted to: " + extract_folder_path)

# walk through raw folder for samples to be extracted
for dirpath, dirnames, filenames in os.walk(raw_folder_path):
    for filename in [f for f in filenames if f.endswith(".lst")]:
        
        # find sample name
        sample_name = os.path.splitext(filename)[0]
        
        # create output directory for extracted vignettes
        sample_outpath = os.path.join(extract_folder_path,sample_name)
        if not os.path.exists(sample_outpath): os.mkdir(sample_outpath)
        
        # get path for .lst file
        fp = os.path.join(dirpath, filename)
 
        # read header of .lst file
        header = pandas.read_csv(fp, sep='|', skiprows=1, nrows=65)
        hd = list(header["num-fields"])
        
        # read .lst file and apply header
        meta = pandas.read_csv(fp, sep='|', skiprows=67, header=None)
        meta.columns = hd
        
        # cheat for loading collage file
        loaded_cp = "not_loaded"
        
        # loop through meta data to extract and save individual vignettes 
        for id in meta["id"]:  
            i = id-1
            
            # find collage name and path
            collage_filename = meta["collage_file"][i]
            cp = os.path.join(dirpath, collage_filename)
           
            # load collage image if not already loaded         
            if not cp == loaded_cp: 
                collage = cv2.imread(cp)
                loaded_cp = cp
           
            # extract vignette
            img_sub = collage[meta["image_y"][i]:(meta["image_y"][i]+meta["image_h"][i]), meta["image_x"][i]:(meta["image_x"][i]+meta["image_w"][i])]
            
            # save vignette
            vp = os.path.join(sample_outpath, sample_name+"_vign"+f"{id:06d}"+".png") 
            cv2.imwrite(vp, img_sub)
        
        # print name of extracted sample when finished
        print("Vignettes extracted: " + sample_name)
```