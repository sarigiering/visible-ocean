+++
title = "From FlowCam to EcoTaxa"
date = 2020-12-04T19:15:08Z
draft = false
tags = ["particles", "FlowCam", "vignette", "extraction", "EcoTaxa", "classification"]
categories = []
featured_image = "img/FlowCam-to-EcoTaxa-1.jpg"
+++

So, you have finished lab work and now have an amazing dataset of images taken by FlowCam. What should you do with the images now?

There are typically two steps that you want to carry out now:

1.   Take particle measurements.
2.   Identify the individual particles.

You can do the above in the software that comes shipped with the FlowCam: VisualSpreadsheet. It is a great software, but you may decide that you want more flexibility. For example, you might want to use a different particle detection routine to measure the particles, or you may want to use a different programme for the image classification.
{{< figure src="/img/FlowCam-to-EcoTaxa-2.jpg" caption="Example of image sorting and classification using EcoTaxa">}}

For semi-automated particle classification, I am a big fan of
<a class="link" href="https://ecotaxa.obs-vlfr.fr/" target="_blank">EcoTaxa</a>.
	EcoTaxa is a web application developed and hosted at 
<a class="link" href="http://www.obs-vlfr.fr/" target="_blank">l'Observatoire Océanologique de Villefranche-sur-Mer</a>.
It allows you to upload your images, carry out some automated presorting, and offers a simple interface for sorting. Because of EcoTaxa's clear interface, the classification process is fast and intuitive. The web-based approach allows you to sort from anywhere (perfect in the new age of home office!), and you can easily share with collaborators and get their input on your suggested classification. Even better, EcoTaxa is community driven and they are ever improving the algorithms.

> The big challenge is now to upload your data onto EcoTaxa!

## Sample file requirements for EcoTaxa
The files and folder structure that FlowCam saves is very different to the one required by EcoTaxa. 

* *FlowCam*: The vignettes are typically saved as part of a collage.
* *EcoTaxa*: All vignettes have to be saved as individual files.

EcoTaxa further asks for **a specific table** to upload the data. Besides informing about the metadata, this table includes the particle measurements. These measurements are critical as the semi-supervised classification algorithm uses these measurements for the classification predictions. Unfortunately, the measurements that FlowCam's VisualSpreadsheet provides are very different to those that EcoTaxa asks for. 

But worry not! We have a solution!

![MorphoCut logo](/img/morphocut_logo.png)

Luckily, I know just the right person to ask for help: Simon-Martin Schroeder, who has helped to implement the deep learning feature in EcoTaxa. More relevant, he developed
<a class="link" href="https://morphocut.readthedocs.io/en/latest/index.html" target="_blank">MorphoCut</a> - an image processing pipeline.

Honestly, writing a MorphoCluster pipeline looks super complicated, but don't be put off by the code. I will point out the important steps.

# The MorphoCut Pipeline

### Vignette extraction
In a <a class="link" href="/posts/extract-individual-particle-images-from-flowcam">previous post</a>,
I shared code that allows you to extract the inividual vignettes for flexible post-processing. We have integrated this code into MorphoCluster to extract the vignettes and produce the table that is required to upload the images into EcoTaxa. 

### Particle measurements
For the particle measurements, you have two options:

1.  Use the particles as detected by VisualSpreadsheet.
2.  Write your own particle detection routine.

Option 1 is by far the easier option and will most likely be sufficient for your purposes. So, this option is the one I will explain here.

Particle measurements include parameters such as particle area, length, height, perimeter, etc. To carry out such measurements, you first need to decide which pixel of an image belongs to the particle. There are several routines - some simple, some sophisticated - that make that decision for you. Each pixel of the image is then assigned as either as 'particle' or 'background'.

System | Background | Particle
:---:|:---:|:---:
logical   | FALSE   | TRUE
binary   |  0  | 1   
mask   | black | white   

This binary system makes it easy to produce a visual representation of what is considered particle: the 'mask'.

During your sample analysis, FlowCam automatically generated a mask for each of the particles and saved them as a collage of binary images ("<sample_name>_??????_bin.tif"). If you are happy with the original particle detection by FlowCam, we can simply use these binary vignettes to calculate our particle characteristics and generate the EcoTaxa table.

{{< figure src="/img/particle_detection.jpg" caption="Example of a FlowCam image ('vignette') and corresponding mask">}}

### Size filter (Awesome!)
If you are like me, you want to make sure you image all particles regardless of how small they are. While you may want to include these small particles when calculating size spectra, they have a downside: They are often so small that they are very difficult to identify, and there are lots of them! My most recent FlowCam dataset contained ~490,000 vignettes, most of which are small little blobs. I am interested in identifying only the largest images, like the image of *Fragilariopsis* above.

The code below allows to only extract vignettes that fullfill a custom criterium. In my case, I chose to retain only particles that have a convex area with &ge;1500 px. The extracted dataset contains only ~3000 particles. 

(In the code below, the size filter is commented out. You can change the threshold to match your purpose or use a different criterium.)

## Setup your directory for extraction
I suggest, you set up the folder as described in my
<a class="link" href="/posts/extract-individual-particle-images-from-flowcam">previous post</a>.
Copy all sample folders that you want to upload onto EcoTaxa in a folder; e.g. `~/<Project_name>/<raw>`. I named it "raw", but you can name it anything you want.


Copy-paste the code below into Python (e.g. Spyder). You should not have to change anything in the code, though you may have to install some of the packages. Simply run the entire code. When prompted select the `<raw>` folder.

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

After extraction, you will find a new folder ("morphocut") in your project folder containing a zipped subfolder with (1) the extracted vignettes and (2) the EcoTaxa table. You should be able to upload this folder straight to EcoTaxa.

Note, in the current version of MorphoCut, all samples are extracted into one single folder. We hope to change this in future versions.

```
.
...
├── <Project_name>
|   ├── <raw>
|   └── morphocut
|   |   ├── ecotaxa.zip
|   |   |   ├── ecotaxa_export.tsv
|   |   |   ├── *_1.jpg
|   |   |   ├── *_2.jpg
|   |   |   ...
...
```

## Code

The code is written in python and you currently need the developer version of MorphoCut.

```py
# Python 3.7
pip install -U git+https://github.com/morphocut/morphocut.git
```

The principle behind the code is a bit complicated and it takes a moment to get your head around it. In simple terms, you write a 'list of actions' that you want to carry out - it is like a generic instructions. Similar to general instructions for driving a car: [1] *open door*, [2] *sit down*, [3] *close door*, [4] *arrange mirrors*, [...]. When you run the code, the computer follows the instructions and applies them on each image/object. The code has the neat aspect that it immediately deletes the bits of data/image/etc that are not needed anymore. It basically tidies up and 'leaves the workbench clean', which makes the image processing much faster.

For more details, check out the <a class="link" href="https://morphocut.readthedocs.io/en/latest/" target="_blank">MorphoCut user guide</a>.

```py
import os, os.path
import tkinter as tk
from tkinter import filedialog
from morphocut.core import Pipeline
from morphocut.file import Find
from morphocut.integration.flowcam import FlowCamReader
from morphocut.image import RGB2Gray, ImageProperties
from morphocut.stream import Filter, TQDM
from morphocut.str import Format
from morphocut.contrib.ecotaxa import EcotaxaWriter
from morphocut.contrib.zooprocess import CalculateZooProcessFeatures

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
        # [Stream] Find .lst files in input path
        lst_fn = Find(raw_folder_path, [".lst"])

        # [Stream] Read objects from a .lst file
        obj = FlowCamReader(lst_fn)
           
        # Extract object image and convert to gray level
        img = obj.image
        img_gray = RGB2Gray(img, True)
        
        # Extract object mask
        mask = obj.mask

        # Extract metadata from the FlowCam
        object_meta = obj.data

        # Construct object ID
        object_id = Format("{lst_name}_{id}", lst_name = obj.lst_name, _kwargs = object_meta)
        object_meta["id"] = object_id

        # Calculate object properties (area, eccentricity, equivalent_diameter, mean_intensity, ...). See skimage.measure.regionprops.
        regionprops = ImageProperties(mask, img_gray)
        
        # ---- Size filter ----
        # [Stream] Only keep large images
        #Filter(regionprops['convex_area']>1499)
        
        # Append object properties to metadata in a ZooProcess-like format
        object_meta = CalculateZooProcessFeatures(regionprops, object_meta)
               
        # [Stream] Here, the images and EcoTaxa table are saved and zipped.
        EcotaxaWriter(
            os.path.join(output_path, "EcoTaxa.zip"),
            [
                # The original RGB image
                (Format("{object_id}.jpg", object_id = object_id), img),
            ],
            object_meta = object_meta,
        )

        # Progress bar
        TQDM(object_id)

p.run()# Requires MorphoCut developer version.
```

