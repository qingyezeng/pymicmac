# pymicmac

pymicmac provides a python interface for MicMac workflows execution and distributed computing tools for MicMac. pymicmac uses pycoeman (Python Commands Execution Manager) (https://github.com/NLeSC/pycoeman) which also provides CPU/MEM/disk monitoring.

MicMac is a photogrammetric suite which contains many different tools to execute photogrammetric workflows.
In short, a photogrammetric workflow contains at least:

 - (1) tie-points extraction: extraction of key features in images and cross-match between different images to detect tie-points (points in the images that represent the same physical locations).

 - (2) Bundle block adjustment: estimation of camera positions and orientations and of calibration parameters.

 - (3) Dense image matching: 3D projection of image pixels to produce the dense point cloud. The 3D points are back projected in the images to correct for projective deformation. This creates a metrically correct True-orthophoto.

In MicMac (1) is usually done with Tapioca, (2) is done with (Tapas) and (3) is done with Malt, Tawny and Nuage2Ply.

pymicmac provides the tool `micmac-run-workflow` to run photogrammetric workflows with a sequence of MicMac commands. The tool uses the sequential commands execution tool of pycoeman which is configured with a XML configuration file that defines a chain of MicMac commands to be executed sequentially. During the execution of each command the CPU/MEM/disk usage of the MicMac-related processes is monitored. The tool can be configured to run a whole photogrammetric workflow at once, or to run it split in pieces (recommended), for example by (1) tie-points extraction, (2) bundle block adjustment and (3) dense image matching.  More information in [Instructions](#instructions) section.

In section [Large image sets](#large-image-sets) we provide some tips on how to use MicMac and pymicmac for processing large image sets using distributed computing for (1) the tie-points extraction and (3) the dense image matching, and tie-points reduction for (2) the bundle block adjustment.

A step-by-step tutorial is also available in [Tutorial](https://github.com/ImproPhoto/pymicmac/tree/master/docs/TUTORIAL.md).

## Installation

Clone this repository and install it with pip (using a virtualenv is recommended):

```
git clone https://github.com/ImproPhoto/pymicmac
cd pymicmac
pip install .
```

or install directly with:

```
pip install git+https://github.com/ImproPhoto/pymicmac
```

Python dependencies: pycoeman and noodles (see https://github.com/NLeSC/pycoeman and https://github.com/NLeSC/noodles for installation instructions)

Other python  dependencies (numpy, tabulate, matplotlib, lxml) are automatically installed by `pip install .` but some system libraries have to be installed (for example freetype is required by matplotlib and may need to be installed by the system admin)

For now pymicmac works only in Linux systems. Requires Python 3.5.

## Instructions

The tool `micmac-run-workflow` is used to execute entire photogrammetric workflows with MicMac or portions of it. We recommend splitting the workflow in three pieces: (1) tie-points extraction, (2) bundle block adjustment and (3) dense image matching. Each time the tool is executed, it creates an independent execution folder to isolate the processing from the input data. The tool can be executed as a python script (see example in `tests/run_workflow_test.sh`) or can be imported as a python module (see examples in `tests/run_tiepoint_detection_example.py`, `tests/run_param_estimation_example.py` and `tests/run_matching_example.py`). Which MicMac commands are executed is specified with a XML configuration file.

### Workflow XML configuration file

The Workflow XML configuration file format is the sequential commands XML configuration file format used by pycoeman (https://github.com/NLeSC/pycoeman). For pymicmac, usually the first tool in any Workflow XML configuration file links to the list of images. So, we can use `<requirelist>` to specify a file with a list of images. Next, some XML examples:

- tie-points extraction:
```
<SeqCommands>
  <Component>
    <id> Tapioca </id>
    <requirelist>list.txt</requirelist>
    <command> mm3d Tapioca All ".*jpg" -1 </command>
  </Component>
</SeqCommands>
```

- Bundle block adjustment
```
<SeqCommands>
  <Component>
    <id> Tapas </id>
    <requirelist>list.txt</requirelist>
    <require>tie-point-detection/Homol</require>
    <command> mm3d Tapas Fraser ".*jpg" Out=TapasOut </command>
  </Component>
</SeqCommands>
```

- Dense image matching:
```
<SeqCommands>
  <Component>
    <id> Malt </id>
    <requirelist>list.txt</requirelist>
    <require> param-estimation/Ori-TapasOut</require>
    <command> mm3d Malt GeomImage ".*jpg" TapasOut "Master=1.jpg" "DirMEC=Results" UseTA=1 ZoomF=1 ZoomI=32 Purge=true </command>
  </Component>
  <Component>
    <id> Nuage2Ply </id>
    <command> mm3d Nuage2Ply "./Results/NuageImProf_STD-MALT_Etape_8.xml" Attr="1.jpg" Out=1.ply</command>
  </Component>
</SeqCommands>
```

Following the examples above, we could execute a whole photogrammetric workflow with:
```
micmac-run-workflow -d /path/to/data -e tie-point-detection -c tie-point-detection.xml
micmac-run-workflow -d /path/to/data -e param-estimation -c param-estimation.xml
micmac-run-workflow -d /path/to/data -e matching -c matching.xml
```

IMPORTANT: all the specified files and folder with the `<require>` and `<requirelist>` as well as the actual files listed in the `<requirelist` must be provided with relative paths to the folder where all the data is. So, the data is in `/path/to/data`, and all the files and folders specified in the `<require>` and `<requirelist>` are relative to this this folder.

### Monitoring

pycoeman (the tool used by pymicmac to run the commands) stores the log produced by each command in a .log file. Additionally, it stores a .mon and a .mon.disk with the CPU/MEM/disk monitoring. pycoeman has tools to get statistics and plots of the monitor files (see https://github.com/NLeSC/pycoeman). pymicmac has some tools to analyze the logs of several MicMac commands: `micmac-tapas-log-anal`, `micmac-redtiep-log-anal`, `micmac-redtiep-log-anal`, `micmac-campari-log-anal`, `micmac-gcpbascule-log-anal`, `micmac-gcpbascule-log-plot`, `micmac-campari-log-plot`.

## Large image sets

For the (1) tie-points extraction and (3) dense image matching the processing can be easily enhanced by using distributed computing (clusters or clouds). The reason is that the processes involved can be easily split in independent chunks (in each chunk one or more images are processed). For the (2) bundle block adjustment, this is not the case since the involved processes usually require having data from all the images simultaneously in memory. In this case, we propose to use tie-points reduction to deal with large image sets.

For more information about distributed computing and tie-points reduction, see our paper (in preparation).

### Distributed computing

Some parts of the photogrammetric workflow, namely the tie-points extraction and the dense image matching, can be boosted by using distributed computing systems since the involved processes can be divided in chunks which are independent to process.

For example, the Tapioca tool (tie-points extraction) first extracts the features for each image and then cross-matches the features between image pairs. The distributed computing solution that we propose is to divide the list of all image pairs in chunks where each chunk can be processed independently (though they may read sometimes the same images). The results from each chunk processing need to be combined.

We use the parallel commands execution tools of pycoeman. The various parallel/distributed commands are specified in a XML configuration file which is similar to the Workflow XML configuration file. An example XML configuration file follows. In this case, we have divided Tapioca processing in two chunks. Each chunk processes the half of the image pairs:

```
<ParCommands>
  <Component>
    <id>0_Tapioca</id>
    <requirelist>DistributedTapioca/0_ImagePairs.xml.list</requirelist>
    <require>DistributedTapioca/0_ImagePairs.xml</require>
    <command>mm3d Tapioca File 0_ImagePairs.xml -1</command>
    <output>Homol</output>
  </Component>
  <Component>
    <id>1_Tapioca</id>
    <requirelist>DistributedTapioca/1_ImagePairs.xml.list</requirelist>
    <require>DistributedTapioca/1_ImagePairs.xml</require>
    <command>mm3d Tapioca File 1_ImagePairs.xml -1</command>
    <output>Homol</output>
  </Component>
</ParCommands>
```

In [Implemented MicMac Tools](#implemented-micmac-tools) we detail the current MicMac tools for which we have implemented a distributed version. The tool `coeman-mon-plot-cpu-mem` of pycoeman can be used to get a plot of the aggregated CPU and MEM usage.

#### Implemented MicMac Tools

##### Tapioca

The tools `micmac-disttapioca-create-pairs`, `micmac-disttapioca-create-config`, `micmac-disttapioca-combine` together with the parallel commands execution tools of pycoeman are used to run Tapioca in distributed computing systems.

First, in order to run this tool we need a image pairs file. This file list the image pairs that need to be considered when running Tapioca. This is very helpful to avoid running Tapioca with every possible image pair. If you do not have a image pairs file, you can use the tool `micmac-disttapioca-create-pairs` to create a image pairs file with every possible image pair.

Second, we use the `micmac-disttapioca-create-config` to split the image pairs XML file in multiple chunks and to create a XML configuration file suitable for pycoeman:

```
micmac-disttapioca-create-config -i [input XML image pairs] -f [folder for output XMLs and file lists, one for each chunk] -n [number of image pairs per output XML, must be even number] -o [XML configuration file]
```

Now, you are ready to run this distributed tool in any of the available hardware systems that are supported by pycoeman (see https://github.com/NLeSC/pycoeman). For this, use the tools `coeman-par-local`, `coeman-par-ssh` or `coeman-par-sge`.

After the distributed Tapioca has finished you will have to combine all the outputs from the different chunks. Use `micmac-disttapioca-combine` to join them into a final Homol folder

```
micmac-disttapioca-combine -i [folder with subfolders, each subfolder with the results of the processing of a chunk] -o [output combined folder]
```

##### Matching (Malt, Tawny, Nuage2Ply)

The tool `micmac-distmatching-create-config` together with the parallel commands execution tool of pycoeman is used to run the matching (point cloud generation) in distributed computing systems.

The algorithm used in the tool `micmac-distmatching-create-config` is restricted to aerial images and when the orientation provided by the step 2 of the photogrammetric workflow is in a cartographic reference system. From the estimated camera positions and assuming that the Z direction in which the pictures are taken is always the same and pointing to the ground, the tool computes the XY bounding box that includes all the XY camera positions. The bounding box is divided in tiles like shown in this illustrative example:

![exampledistmatching](docs/distmatching_example.png)

Each tile can be processed by an independent process. For each tile the cameras(images) whose XY position lays within the tile are used. This list is extended to guarantee a minim of 6 images per tile with nearest neighbors.

The tool `micmac-distmatching-create-config` splits the matching of a large list of images into tiles and create a XML configuration file suitable for pycoeman.

```
micmac-distmatching-create-config -i [orientation folder] -t [Homol folder] -e [images format] -o [XML configuration file] -f [folder for extra tiles information] -n [number of tiles in x and y]
```

Now, you are ready to run this distributed tool in any of the available hardware systems that are supported by pycoeman (see https://github.com/NLeSC/pycoeman). For this, use the tools `coeman-par-local`, `coeman-par-ssh` or `coeman-par-sge`.

### Tie-points reduction

Add a tie-points reduction component in the chain for parameters estimation.

Two tools can be used for this purpose: `RedTieP` and `OriRedTieP`. The first one requires to run the tool `NO_AllOri2Im` before and the second requires to run the tool `Martini` before.

For examples, see `tests/param-estimation_reduction.xml` and  `tests/param-estimation_orireduction.xml`

When a tie-points reduction is used with either of the available tools, the tool `micmac-homol-compare` can be used to compute the reduction factors.

Note that after running the tie-points reduction tools, the Homol folder has to be changed (see the examples).
Also note that when running `RedTieP`, it is possible to use parallel execution mode together with the tool `micmac-noodles`. See the example in `tests/param-estimation_reduction.xml`.
