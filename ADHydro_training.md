Instructions for creating a mesh from partially processed data.

See [Input Creation Steps](ADHydro_input_creation_instructions.md) for details on the steps leading up to this point. (Section 1, steps 1 - 65)

These workflows make a few assumptions about the directory struture of ADHydro input files. Typically, a top level directory for each mesh/map is used to setup and run ADHydro.  The following structure is partially established and will be used throughout this training:

```
map_dir/
|-- ASCII
|   |-- mesh.1.chan.ele
|   |-- mesh.1.chan.node
|   |-- mesh.1.chan.prune
|   |-- mesh.1.chan.z
|   |-- mesh.1.edge
|   |-- mesh.1.ele
|   |-- mesh.1.geolType
|   |-- mesh.1.landCover
|   |-- mesh.1.link
|   |-- mesh.1.neigh
|   |-- mesh.1.node
|   |-- mesh.1.poly
|   |-- mesh.1.soilType
|   |-- mesh.1.z
|   |-- mesh.node
|   `-- mesh.poly
|-- ArcGIS
|   |-- intersect_wtrbodies_final.dbf
|   |-- intersect_wtrbodies_final.prj
|   |-- intersect_wtrbodies_final.shp
|   |-- intersect_wtrbodies_final.shx
|   |-- mesh_catchments.dbf
|   |-- mesh_catchments.prj
|   |-- mesh_catchments.qpj
|   |-- mesh_catchments.shp
|   |-- mesh_catchments.shx
|   |-- mesh_streams.dbf
|   |-- mesh_streams.prj
|   |-- mesh_streams.shp
|   |-- mesh_streams.shx
|   |-- mesh_waterbodies.dbf
|   |-- mesh_waterbodies.prj
|   |-- mesh_waterbodies.shp
|   |-- mesh_waterbodies.shx
|   |-- wtrbodies_streams_intersect_final.dbf
|   |-- wtrbodies_streams_intersect_final.prj
|   |-- wtrbodies_streams_intersect_final.shp
|   `-- wtrbodies_streams_intersect_final.shx
|-- TauDEM
|   |-- projected
|   |   |-- projected.tif
|   |   `-- projected.tif.aux.xml
|   |-- projectednet.dbf
|   |-- projectednet.prj
|   |-- projectednet.shp
|   `-- projectednet.shx
|-- analysis
|   |-- outline.cpg
|   |-- outline.dbf
|   |-- outline.prj
|   |-- outline.shp
|   `-- outline.shx
|-- data
|   |-- geology
|   |-- nlcd_2016
|   `-- soil
|       |-- SSURGO
|       `-- STATSGO
|-- drain_down
|   |-- display.nc
|   |-- geometry.nc
|   |-- parameter.nc
|   |-- state.nc
|   `-- superfile.ini
|-- forcing
|   |-- forcing_2020-10-2_2020-10-2.nc
|-- mesh_massage
|   |-- display.nc
|   |-- geometry.nc
|   |-- parameter.nc
|   |-- state.nc
|   `-- superfile.ini
`-- simulation
    |-- display.nc
    |-- geometry.nc
    |-- parameter.nc
    |-- state.nc
    |-- state.nc.maxDepth.txt
    `-- superfile.ini
```
The TauDEM directory contains the TauDEM generated stream network from previous steps, and the ArcGIS directory contains topologically corrected mesh catchments, streams, and waterbodies.  From these, the following steps will begin populating the ASCII directory.

TODO: preamble docker interactive
TODO: add a section at the end on how to run ADHydro with slurm/srun/sbatch
TODO: Later: get docker run to work right with tools

For these steps, you will be running an interactive `adhydro-tools` container, and entering commands on the terminal within that container.

Start the container with the following

TODO extract data/docker build

``docker run -it -v `pwd`/training_data:/data adhydro-tools``

1.  Use a python script to create input files for triangle
    from these previously created shapefiles.

    1. Run the script create_triangle_files.py.

    `create_triangle_files.py <map_dir>`

    From the docker terminal:

    `python /scripts/preprocessing/create_triangle_files.py /data`

    Now `/data/ASCII` should have the following files.  These are simply ascii formatted geometry descriptions of the mesh catchments, streams, and waterbodies.  These are specifically formatted to be inputs to the triangle mesh creation program.

    - mesh.1.link
    - mesh.node
    - mesh.poly

    Verify with `ls /data/ASCII`

2. Now you will run triangle to generate the mesh.  In the ASCII directory:
    `cd /data/ASCII`

    `triangle -pqAjenV mesh.poly`

    or with docker

    ``docker run -v `pwd`/data:/data adhydro_tools cd /data/ASCII && triangle -pqAjenV mesh.poly``


    1.  There may be lots of warnings about duplicate vertex and
        endpoints of segment are coincident. These can be ignored. If
        your dataset is big it can take triangle a long time just to
        print these out. If you want, you can edit the triangle source
        code to comment out these print statements and recompile.
    2.  If there are overlaps and gaps in the mesh created by a topology
        failure described in step 70 then triangle might crash or go
        into an infinite loop generating tiny triangles in those problem
        areas. The only way I found to get around this is to run
        triangle in the debugger, break in to find what x, y coordinates
        it is getting hung up on and then find that location in ArcGIS
        and fix the gap or overlap by hand.

3. You can analyze the mesh with the program adhydro_mesh_check.
    It will report problems with small triangles and triangles that have
    no catchment label.

    In another terminal/window run the following command to use the adhydro utility

    `cd training_dir`

    ``docker run -v `pwd`/training_data:/data adhydro -c "adhydro_mesh_check /data/ASCII/"``

    1.  Triangles with no catchment label shouldn\'t occur. They can
        occur if a catchment polygon is bisected by streams, but because
        of the split polygons step there shouldn\'t be any of those
        situations. If there are triangles with no catchment label you
        will need to investigate why.
    2.  Small triangles usually result from sharp angles in the
        catchment polygons. This can be fixed by hand in ArcGIS.  See [Input Creation Steps](ADHydro_input_creation_instructions.md) for details.  For this example, we won't worry
        about these.

4. Now you will use a python script to extract the z coordinate from
    the DEM for each point in the mesh.
    Switch back to the terminal running the `adhydro-tools` container.
    In this container we will run the following command.

    `python /scripts/preprocessing/create_z_file.py /data`

    This uses the `.node` file in the ASCII directory and the original non-pit-filled DEM in the TauDEM directory for input. It creates a `.z` file in the ASCII directory.

    `ls /data/ASCII` to verify the `.z` file was created.

    TODO `.z` file contains <> infomation.

5.  Run the program adhydro_channel_preprocessing. This will
    create three files: a `.chan.ele` file that contains the channel
    elements, a `.chan.node` file that contains the x,y coordinates of the
    channel nodes, and a `.chan.prune` file that contains pruned reach
    codes each paired with an unpruned reach code that the pruned reach
    code flows downstream into. The `.chan.node` file is in the same
    format as a triangle `.node` file so you can use the same script to
    generate the z coordinates files later.

    Switch to the other terminal, or in a new terminal/window run the following docker command to
    execute the adhydro channel preprocessing utility.

    ``docker run -v `pwd`/training_data:/data adhydro -c "adhydro_channel_preprocessing /data"``

    1.  We found that the values in the shapefiles for link type were
        not consistent across data sources. In Wyoming the values were
        words: \"Ice Mass\", \"LakePond\", \"SwampMarsh\", etc. In
        Colorado the values were numbers: \"378\" for Ice Mass, \"390\"
        for LakePond, \"466\" for SwampMarsh, etc. It seems likely that
        any time you process a mesh in a new political unit you will
        need to figure out what they use for link type and update the
        code of adhydro_channel_preprocessing. You will get an error
        message if the shapefiles use a link type string that is
        unrecognized.

    Now we will create the channel `.z` files from the `.chan.node` file and the original
    non-pit-filled DEM. The `create_z_file.py` script has a flag for processing channels.

    Back in the adhydro-tools container terminal, run the same script as before with the channel flag `-c`

    `python /scripts/preprocessing/create_z_file.py -c /data`

    This generates the `.chan.z` file in the ASCII directory.

    `ls /data/ASCII`

#Parameter Data

This section is covered in detail in See [Input Creation Steps](ADHydro_input_creation_instructions.md), section IV: Process parameter data

In this section you will assign parameters like soil and vegetation type to mesh elements.

##Download data

1.  Before creating the parameter files for a given mesh you need to
    download several kinds of publicly available source data. The
    downloaded source data can be used for processing just one mesh and
    then thrown away, or you can save it to use for multiple meshes. If
    you want to use it for multiple meshes you need to make sure that
    you download data that covers the complete area of all of the meshes
    you will use it for.

    1. Create a data directory for your project, \<project path\>/data, and
    create the following subdirectories: FIXME <project path>

        1.  \<project path\>/data/soil
        2.  \<project path\>/data/soil/SSURGO
        3.  \<project path\>/data/soil/STATSGO
        4.  \<project path\>/data/geology
        5.  \<project path\>/data/nlcd_2016 -- FIXME in general

2.  You will now download SSURGO and STATSGO data by lat/long bounding
    rectangle.

    1.  First we will generate a bounding box for the mesh using an adhydro script.  This will be used in the next step.
    In the `adhydro-tools` container, run the following command

    `mkdir /data/analysis`

    `python /scripts/gis/mesh_outline.py -o /data/analysis -b 1000 /data/ArcGIS/mesh_catchments.shp`

    This same script can be used to simply report the extents of the mesh by using the `-x` flag.

    `/scripts/gis/mesh_outline.py -x /data/ArcGIS/mesh_catchments.shp`

    2. Go to the [USDA Web Soil Survey](https://websoilsurvey.sc.egov.usda.gov/App/WebSoilSurvey.aspx)


    3. On the left, under Area of Interest, click "Import AOI"

    4. Click "Create AOI from Shapefile".  Upload the outline.shp outline.shx, outline.prj files from `training_data/analysis`. Click "Set AOI"

    5. Once the AOI loads, click the "Download Soils Data" tab on top of the map.

    6. You should be on a "Your AIO (SSURGO)" section.
        1. Click "Create Download Link" near the bottom right of the section.
           Once it completes loading, the panel now has a "Download Link".  Click it to download the SSURGO data, then move the downloaded zip file to `training_data/data/soils/SSURGO/`

    7. Click the "U.S. General Soil Map (STATSGO2)" section.
       Find the state/states that your area of interest is in, in this case MD, and download the zip/s.
       Move the STATSGO downloaded data to `training_data/data/soils/STATSGO`

3. Now you will extract the archives and merge the spatial data needed
    for building ADHydro inputs. Run the script `prepare_soil_data.py`
    with the following command in the `adhydro-tools` container terminal.

    `python /scripts/preprocessing/prepare_soil_data.py -x /data/data/soil/`

    Next is a quick workaround that is needed to get the processing script later to work correctly.
    The AOI download packages the zip structure differently than the soil survey area, so we need to key the area of interested as if it were a soil survey area.

    `mv /data/data/soil/SSURGO/wss_aoi_<tab> mv /data/data/soil/SSURGO/MD005` FIXME required???
    When typing the above command, use tab complete to help

4. Download the latest National Land Cover Database (NLCD) from
    <http://www.mrlc.gov/>.
    We can use the mesh extents discovered previously to download only a subset of the NLCD.
    The following url is constructed from these extents, rounded up slightly ensure we have proper coverage
    accross projections. You can use the controls in the data download box to adjust the extent for other meshes.

    https://www.mrlc.gov/viewer/?downloadBbox=39.28,39.35,-76.80,-76.71

    1. Follow the above link, in the Data Download box, select "Land Cover" and "2016 Land Cover ONLY".
    2. Enter your e-mail in the e-mail box, you should receive a download link momentarily if the extent isn't too large.  Alternatively, you CAN download the conus dataset and use it.  The follow steps apply to either case.
    3. Move the downloaded data to `training_data/data/nlcd_2016` (and extract if using the subset)

    4. Create a virtual raster layer.
       From the `adhydro-tools` container terminal, run the following
       ```
       cd /data/data/nlcd_2016
       gdalbuildvrt nlcd.vrt NLCD_2016_Land_Cover_L48_*.tiff
       cd /data
       ```

5. Download Geologic Units from
    <http://mrdata.usgs.gov/geology/state/>. Use the Conterminous US
    state geology <https://mrdata.usgs.gov/geology/state/geol_poly.zip>.

6. Extract this dataset to training_data/data/geology

7. You are now finished downloading the source data that you will need.
    The steps after this need to be done for each mesh even if you had
    the downloaded source data saved from a previous mesh.

##Process Data

1.  Run `parameter_preprocessing.py`
    In the `adhydro-tools` container terminal run the following command.  A quick note on the `-m` flag.
    This is the central meridian of the sinusoidal projection used for ADHydro meshes.  This meridian is selected in the early steps of the mesh creation.  It can be found in the ArcGIS .prj files if needed.  For the Dead Run mesh, the value is `-76`

    `python /scripts/preprocessing/parameter_preprocessing.py /data -m -76`


    NOTE: For large meshes, this can take a while, about 8 hours for the
    Upper Colorado Basin -- 9,186,658 elements -- running on a 16
    core machine with 126 GB memory. It used approximately 80 GB of
    memory at peak.

    This script will create a lot of intermediate files. The file names
    begin with "element\_". For example,
    "element_cokey_data.parquet.0-X", "element_coord_data.parquet.0-X".
    These files can be used to recover from a crash by
    restarting partway through the run using the saved intermediate
    values. However, this isn't well documented. This
    functionality is really only useful on really large meshes that take
    a very long time to process. So you can delete these files if you
    want.

    The three files that you need that are created by this processing
    are "mesh.1.geolType", "mesh.1.landCover", and "mesh.1.soilType".

    Verify these files exist:

    `ls /data/ASCII | grep mesh`

## Precondition the mesh with ADHydro

1.  At this point, you should have files ready to use as ASCII input
    files to the ADHydro simulation code. The following files will be
    read in by ADHydro. These are also listed in the example superfile.

    1.  mesh.1.node
    2.  mesh.1.z
    3.  mesh.1.ele
    4.  mesh.1.neigh
    5.  mesh.1.landCover
    6.  mesh.1.soilType
    7.  mesh.1.geolType
    8.  mesh.1.edge
    9.  mesh.1.chan.node
    10. mesh.1.chan.z
    11. mesh.1.chan.ele
    12. mesh.1.chan.prune

2.  Read ASCII files with mesh massage

    TODO summary Mesh Massage Conifguration Table

    This configuration editing step can be done without using Docker.  Simply create the directories and copy the `example_superfile.ini` file from <FIXME WHERE???>.

    Create a directory called `mesh_massage` and copy the ADhydro example superfile to this directory.  You can rename it to simply superfile.ini if you prefer.

    `cp /adhydro/example_superfile.ini mesh_massage/superfile.ini`

    This file will confiure the adhydro code. Start by with line 13, it should look like

    `evapoTranspirationInitDirectoryPath = /adhydro/HRLDAS-v3.6/Run`

    Uncomment line line 33 `initializeFromASCIIFiles` and set it to `true`.

    Next set the `ASCIIInputDirectoryPath` to the directory where the mesh files were created in the previous steps, i.e

    `ASCIIInputDirectoryPath = /data/ASCII`

    Comment out line 54, `adhydroInputDirectoryPath` since we are initializing from ASCII first.

    Set the output directory on line 70 to the `mesh_massage` path, i.e.

    `adhydroOutputDirectoryPath = /data/mesh_massage`

    On line 79, set the map projection information.  Uncomment centralMeridianDegrees and set it accordingly.

    `centralMeridianDegrees = -76.0`

    Then uncomment the falseEasting and falseNorthing lines, but you shouldn't need to change these values.

    The mesh_massage doesn't actually do any simulation, but ADHydro requires a referenceDate and currentTime to be set regardless, so uncomment the `referenceDateJulian` variable on line 89 and the `currentTime` on line 99.

    Finally, uncomment line 139, `doMeshMassage` and set it to `true`

    Now run adhydro using this superfile.

    `adhydro superfile.ini`

4.  Create forcing data.
    For this training, we will use National Water Model analysis and assimilation forcings pulled from Google Cloud Platform.  This is provided in the training material under `forcing/data`.

    Use the `adhydro-tools` container terminal to run the following command.  It will resample the gridded forcings and create a single time varying forcing netCDF file that ADHydro can read during simulations.
    First, change to the forcing data directory.

    `cd /data/forcing`

    then

    `python /scripts/forcing/nwm/nwm_2.0_to_adhydro.py -s 2019-07-01:00 -e 2019-12-31:23 -m -76 -c 2 /data /data/forcing/data/`

5.  Do drain-down run.
    Create a directory called `drain_down` and copy the `mesh_massage` superfile to this directory.

    `cp mesh_massage/superfile.ini drain_down/`

    Now we need to turn off the ASCII initialization and use the mesh_massage generated netCDF files.

    Edit line 33, comment the line out with a ; or set the value to `false`.

    Uncomment line 54 and set it to the mesh_massage directory.

    `adhydroInputDirectoryPath     = /data/mesh_massage`

    Uncomment line line 58 and set its value to the forcing file created.

    `adhydroInputForcingFilePath   = /data/forcing/forcing_2020-10-2_2020-10-2.nc` FIXME filename

    Set the output directory on line 70 to the `drain_down` path, i.e.

    `adhydroOutputDirectoryPath = /data/drain_down`

    Since we are actually going to prepare a simulation, we need to set the reference time accordingly.  This should be the julian date of the first available forcing time.

    One way to find this is to use ncdump and copy the first output of the variable
    `JULTIME`.

    `ncdump -v JULTIME forcing_2020-10-2_2020-10-2.nc | grep JULTIME`

    Set line 89 `referenceDateJulian` to this value.  Alternatively, comment out line 89 and set each subsequent date value.

    Leave `currentTime` on line 99 to indicate starting at the beginning of the forcing record.

    Set the drain_down duration, about 2 weeks should be sufficient.
    Uncomment line 102 and set the `simulationDuration` to 2 weeks, in seconds:

    `simulationDuration = 1209600.00`

    Finally, comment out line 139, `doMeshMassage` or set it to `false`, and then uncomment `drainDownMode` on line 133 and set it to true.

    Save this file and then execute adhydro.

    ``docker run --cap-add=SYS_PTrace -v `pwd`/training_data:/data adhydro -c "mpirun -n 8 adhydro /data/drain_down/superfile.ini"``

6. Run simulation
    Make a simulation directory and copy the drain_down superfile to this directory.

    `cp drain_down/superfile.ini simulation/`

    Edit the superfile to configure the simulation.

    Edit the input directory path on line 54 to point to the drain_down location.

    `adhydroInputDirectoryPath     = /data/drain_down`

    Set the output directory on line 70 to the simulation directory.

    `adhydroOutputDirectoryPath     = /data/simulation`

    Uncomment and set the simulation duration on line 102 based on the amount of time, in seconds, you want to simulate (relative to the reference date).  You can run longer than forcing is available, in which case the last known values of forcing are used for the rest of the simulation.  In general, though, set this duration for the duration of available forcing. To run 1 day:

    For one day:

    `simulationDuration  = 86400.0`

    For 180 days:

    `simulationDuration  = 15552000.0`

    Set the outputPeriod on line 106 to an approriate interval, in seconds.

    For hourly output:

    `outputPeriod        = 3600.0`
    
    Disable the drainDownMode on line 133 by commenting or setting to false.

    Run the simulation:

    `adhydro simulation/superfile.ini`

