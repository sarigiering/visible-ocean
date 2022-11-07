+++
title = "Save a pile of images for ExoTaxa"
date = 2022-11-07T12:42:20
draft = false
tags = ["particles", "vignette", "MorphoCut", "EcoTaxa"]
categories = []
featured_image = "img/stack_ecotaxa.png"
+++

We are trying a new workflow to extract particles from holographic images taken by the LISST-Holo and would like to import the images to <a class="link" href="https://ecotaxa.obs-vlfr.fr/" target="_blank">EcoTaxa</a> for manual classification. In a previous post, I already introduced [EcoTaxa and MorphoCut]({{< ref "/posts/from-flowcam-to-ecotaxa" >}}).

The files and folder structure required by EcoTaxa are as follows: 

* *EcoTaxa table*: File names and particle details for every vignette.
* *Vignettes/images*: All vignettes saved as individual files.

All files are saved in a folder and zipped ready for uploading to EcoTaxa.

I found the quickest way to do this is to modify the <a class="link" href="https://morphocut.readthedocs.io/en/latest/index.html/" target="_blank">MorphoCut</a> pipeline I used previously to [extract and save the FlowCam images]({{< ref "/posts/from-flowcam-to-ecotaxa" >}}). For particle measurements, I went very basic and simply applied a threshold value of 120. This value may not be appropriate for your images and you may want to modify this value (or implement a thresholding algorithm) particularly if you are planning to use EcoTaxa's pre-sorting features. For our case, it will do the job as we do not have many vignettes and are planning to sort them manually anyway.

## Code 
The code is written in python and you currently need the developer version of MorphoCut.

```py
# Python 3.7
pip install -U git+https://github.com/morphocut/morphocut.git
```

The principle behind the code is a bit complicated and it takes a moment to get your head around it. In simple terms, you write a 'list of actions' that you want to carry out - it is like a generic instructions. For more details, check out the <a class="link" href="https://morphocut.readthedocs.io/en/latest/" target="_blank">MorphoCut user guide</a>.

For our purpose, simply copy-paste the code below into Python and run it. It will have installed a function that you can call. To 'ExoTaxa' your files, type the following command into Python, and modify the latitude, longitude and dates as appropriate for your data. You will also need to specify the format of the source images ('ext').  

```py
make_ecotaxa_folder( 
			folder_name = "COMICS_Event034",
			lat = "-52.69",
			lon = "-40.125",
			date = "2017-11-16",
			ext = ".bmp")
```	

So, in this example, the zipped folder will be called *EcoTaxa_COMICS_Event034* (the prefix is added automatically). The metadata will be updated to include latitude, longitude and date of all images (in this case they will be 52.69 S, 40.125 W, and 16 Nov 2017). The source images are bitmaps (".bmp"). The fixed metadata works for us as we are looking at vertical profiles. If I find some time in future, I might include an option for more dynamic metadata.

The function will pop a window that will allow you to select the folder that contains the vignettes that you want to zip. It will then save the zipped folder in the parent directory, which you can directly upload to EcoTaxa. (Though remember, the metadata is limited!)

{{< figure src="/img/2022_11_07_popup.png" caption="Example of pop-up window to select source folder">}}

```py
# Note, this code requires the installation of MorhoCut developers version.
# https://morphocut.readthedocs.io/en/stable/introduction.html

# Python 3.7
# pip install -U git+https://github.com/morphocut/morphocut.git
 
# ---- Required packages ----
import os, os.path
import tkinter as tk
from tkinter import filedialog
from morphocut.core import Pipeline, Call
from morphocut.file import Find, Glob
from morphocut.image import ImageProperties, ImageReader
from morphocut.stream import Progress
from morphocut.str import Format
from morphocut.contrib.ecotaxa import EcotaxaWriter
from morphocut.contrib.zooprocess import CalculateZooProcessFeatures

def make_ecotaxa_folder(folder_name, lat = None, lon = None, date = None, ext = ".png"):
    """
    Make ecotaxa table and pack table and all images into folder (zipped) for direct upload into Ecotaxa.
    
    Parameters
    ----------
    folder_name: str
        Name to be used for output folder. E.g. "EcoTaxa_Event1"
    lat: float
        Latitude (with South being negative)
    lon: float
        Longitude (with West being negative)
    date: str of format "YYYY-MM-DD"
        Date of sampling
    ext: str
        Extension of images (e.g. ".bmp", ".png")
    
    Returns
    --------
    Folder
        Saves EcoTaxa table and all images in zipped folder
    
    Note
    -------
    Masks for particles are produced using a threshold of 120. This fixed threshold may not be appropriate for your data.
    """
    
    # prompt for choosing folder
    root = tk.Tk()
    root.attributes('-topmost', 1)
    root.withdraw()
    raw_folder_path = filedialog.askdirectory()
    root.update()
    os.chdir(raw_folder_path)
    
    # make directory for extraction
    output_path = os.path.join(os.path.dirname(raw_folder_path), "morphocut")
    if not os.path.exists(output_path): os.mkdir(output_path)
    
    # print folder names
    print("Selected folder: " + raw_folder_path)
    print("Files will be extracted to: " + output_path)
    
    # MorphoCut pipeline
    if __name__ == "__main__":
    
        with Pipeline() as p:
    
            # [Stream] Find path of .bmp files in input path
            fn = Find(raw_folder_path, [ext])
    
            # --- metadata table ---
            # Extract file path (Corresponds to `for path in glob(pattern):`)
            path = Glob(fn)
            
            # Remove path and extension from the filename
            basename = Call(lambda x: os.path.splitext(os.path.basename(x))[0], path)
            
            thisdict = {
              "id": Format("{object_id}", object_id = basename),
              "lat": lat,
              "lon": lon,
              "date": date,
            }
    
            # --- image processing ---
            # [Stream] Read and open image from path. Note, it's already black-and-white
            img = ImageReader(fn)
                      
            # Make object mask
            mask = img < 120
          
            # Calculate object properties (area, eccentricity, equivalent_diameter, mean_intensity, ...). See skimage.measure.regionprops.
            regionprops = ImageProperties(mask, img)           
       
            # Append object properties to metadata in a ZooProcess-like format
            meta = CalculateZooProcessFeatures(regionprops, thisdict)
            # End of parallel execution
    
            # [Stream] Here, three different versions are written. Remove what you do not need.
            EcotaxaWriter(
                os.path.join(output_path, "EcoTaxa_" + folder_name + ".zip"),
                [
                    # The original RGB image
                    (Format("{object_id}.jpg", object_id = basename), img),
                ],
                object_meta = meta,
            )
    
            # Progress bar
            Progress(fn)
    
        p.run()# Requires MorphoCut developer version.
```