# How to Write a Model Reader
This section guides the reader through the process of writing a model reader by example. It walks through all of the required pieces of a model reader, using the DTM model reader as an example (see Kamodo/kamodo_ccmc/readers/dtm_4D.py), and describes how the current code was created. The DTM is the simplest, most straightforward model reader to understand because there are no file conversions or pre-processing steps required, the data is presented in a single series of files sequenced in time, no function compositions are needed, and there are no other special features to navigate. The DTM model reader script will be analyzed and explained here piece by piece, and compared to other more complex examples along the way. For a list of the docstrings for all of the model readers written to date, go to (LINK TO FILE HERE). 

The design of the model readers in the kamodo_ccmc repository is geared towards static datasets - datasets that do not grow over time. If the model reader is to be used in real-time applications, such as functionalizing data produced by a model running in real-time, the structure below will likely be too inefficient, particularly for model readers requiring pre-processing for pressure level inversions and other custom coordinate system processes. Further research is required to determine the best structure for that case.

## The variable metadata dictionary: model_varnames
The model_varnames dictionary contains all of the metadata for the variables in the model output and all derived variables supported by the model reader. Any variables that the model developers do not want functionalized should simply be excluded from this dictionary. Derived variables may also be included in this dictionary, along with the supporting code in the reader (e.g. see the CTIPe model_varnames dictionary and the supporting function composition logic towards the bottom of the reader, in Kamodo/kamodo_ccmc/readers/ctipe_4D.py). The dictionary for each model reader is always located at the top of the script and has the same name in each script. This is to align with the variable searching logic in the model_wrapper.py script in the Kamodo/kamodo_ccmc/flythrough directory, and greatly simplifies the logic required to provide model-agnostic searching capabilities to the user. See the model_varnames example from the DTM model reader script below.
```py
model_varnames = {'Temp_exo': ['T_exo', 'Exospheric temperature', 0, 'GDZ',
                               'sph', ['time', 'lon', 'lat'], 'K'],
                  'Temp': ['T', 'temperature', 0, 'GDZ', 'sph',
                           ['time', 'lon', 'lat', 'height'], 'K'],
                  'DEN': ['rho', 'Total mass density', 0, 'GDZ', 'sph',
                          ['time', 'lon', 'lat', 'height'], 'g/cm**3'],
                  'MU': ['m_avgmol', 'Mean molecular mass', 0, 'GDZ', 'sph',
                         ['time', 'lon', 'lat', 'height'], 'g'],
                  'H': ['N_H', 'Atomic hydrogen partial density', 0, 'GDZ',
                        'sph', ['time', 'lon', 'lat', 'height'], 'g/cm**3'],
                  'He': ['N_He', 'Atomic helium partial density', 0, 'GDZ',
                         'sph', ['time', 'lon', 'lat', 'height'], 'g/cm**3'],
                  'O': ['N_O', 'Atomic oxygen partial density', 0, 'GDZ',
                        'sph', ['time', 'lon', 'lat', 'height'], 'g/cm**3'],
                  'N2': ['N_N2', 'Molecular nitrogen partial density', 0,
                         'GDZ', 'sph', ['time', 'lon', 'lat', 'height'],
                         'g/cm**3'],
                  'O2': ['N_O2', 'Molecular oxygen partial density', 0, 'GDZ',
                         'sph', ['time', 'lon', 'lat', 'height'], 'g/cm**3']
                  }
```

### Keys
The dictionary object in Python is similar to a list in other languages and is indicated by curly brackets. The 'keys' of the dictionary are typically strings or integers and are followed by colons. In the model_varnames dictionary, the keys are the strings indicating the names of the variables as represented in the data files and are case-sensitive. For data saved in netCDF files, a list of these keys can be obtained by the following code:
```py
from netCDF4 import Dataset
cdf_data = Dataset(file)
cdf_data.variables.keys()
```
Similar methods can be used in other file formats. Collaboration with model developers is typically essential for data stored in binary or compressed binary file formats.

### Variables
The keys in the model_varnames dictionary are followed by a list of items. The first item is the LaTeX representation of the item following the syntax rules of Kamodo (see https://ensemblegovservices.github.io/kamodo-core/notebooks/Syntax/). The representation of variables in Kamodo is somewhat standardized across model outputs in alignment with the typical representation of the variable in literature. For example, density is always indicated as rho with a subscript indicating which density type it is.

The second item in the list is a string containing the full description of the variable, which is included to be clear on what the variable represents. In some cases, this is long enough to split over multiple lines to be compatiable with PEP8 formatting standards.  

This string is followed by an integer, typically 0. Previously, the inclusion of the integer was to aid in using the flythrough function through a direct link to Fortran, but it should now used to indicate whether the variable is a scalar, represented by a zero, or a vector component. Vector components are represented by a non-zero value that corresponds to the other vector components in the dictionary. For example, if the three components of magnetic field and velocity are included in a given model reader, all three components of the magnetic field would be indicated with a 1, and all three components of the velocity would be indicated with a 2. This is to simplify programmatic grouping of the vector components when the new coordinate conversion capability is implemented by Ensemble. The DTM model output only contains scalars, so all of the integers in the model_varnames dictionary are zeros.

### Coordinate system
The coordinate system and type are indicated by the next two items in the list as strings. The first string indicates which of the supported coordinate systems that variable is given in. The chosen string must be identical to one of the strings in the two lists at the top of the utils.py script (in Kamodo/kamodo_ccmc/flythrough/) called astropy_coordlist and spacepy_coordlist. Please see these lists and the coordinate system definitions in the AstroPy and SpacePy documentation websites for more details. The developer is cautioned to be aware of the detailed differences between these coordinate systems and choose accordingly.

For model specific coordinate systems, a subscript is typically added to the LaTeX representation of the variable indicating the model-specific coordinate system. In those cases, the coordinate system to which the variable will be converted to is indicated in these two strings. See the example from the CTIPe model reader below (Kamodo/kamodo_ccmc/readers/ctipe_4D.py). Note that two versions of the density variable are included in the dictionary, one for the variable from the file in the original model-specific coordinate system (pressure level in this case), and a second one for the variable that is converted to the GDZ spherical coordinate system through function composition later in the script. Additional logic is required to deal with the associated complications, as shown in the model readers with model-specific coordinate systems (currently CTIPe, TIE-GCM, WACCM-X, and WAM-IPE).
```py
model_varnames = {'density': ['rho_ilev1', 'total mass density', 0, 'GDZ',
                              'sph', ['time', 'lon', 'lat', 'ilev1'],
                              'kg/m**3'],
                  'density_2': ['rho', 'total mass density', 0, 'GDZ', 'sph',
                                ['time', 'lon', 'lat', 'height'], 'kg/m**3']
                  }
```
The second string after the integer simply indicates whether the coordinate system is cartesian 'car' or spherical 'sph'.

The two coordinate system strings are followed by a list of strings indicating what coordinates the variable depends on. Note that all but the first variable in the DTM model_varnames dictionary at the top of this page depend on four coordinates, while the first depends on three. Since the coordinate system is spherical, the order of the coordinates must be time ('time'), longitude ('lon'), then latitude ('lat'), followed by height (if four coordinates are required). The names of the coordinates should correspond to the coordinate system and type chosen. If the coordinate type is cartesian, the proper order is 'time', 'X', 'Y', 'Z'. Model specific coordinate systems should follow the same convention. The time coordinate should be listed first if the variable is time-dependent, followed by the described order of the coordinate names.

### Units
At the end of the list for each variable is a string representing the units. The syntax of this string follows the typical python operation syntax. Unitless variables are represented with an empty string.

## Logical Structure and Docstring
The following portion of code, referred to as the preamble, is typically copied from reader to reader to ensure identical function between the readers. This code determines the logical structure of the call signature for the model reader, which should be identical between model readers. The import statements vary between readers depending on the specific logic required for each case, but the structure of the preamble is and must be identical across readers. Specifically, a function called MODEL is defined, which returns a class called MODEL, which subclasses Kamodo. The syntax of the init function of the MODEL class is constructed identically across readers as described in the docstring shown below. The Inputs and Returns section of the docstring are identical across readers. These are followed by a Notes section, which has model-specific notes about the custom logic in each model reader. See the Functionalizing Modeled Datasets section for examples on running a model reader.
```py
def MODEL():
    from time import perf_counter
    from glob import glob
    from os.path import basename, isfile
    from numpy import array, unique, NaN, append, transpose, where
    from datetime import datetime, timezone
    from netCDF4 import Dataset
    from kamodo import Kamodo
    import kamodo_ccmc.readers.reader_utilities as RU

    class MODEL(Kamodo):
        '''DTM model data reader.

        Inputs:
            file_dir: a string representing the file directory of the
                model output data.
                Note: This reader 'walks' the entire dataset in the directory.
            variables_requested = a list of variable name strings chosen from
                the model_varnames dictionary in this script, specifically the
                first item in the list associated with a given key.
                - If empty, the reader functionalizes all possible variables
                    (default)
                - If 'all', the reader returns the model_varnames dictionary
                    above for only the variables present in the given files.
            filetime = boolean (default = False)
                - If False, the script fully executes.
                - If True, the script only executes far enough to determine the
                    time values associated with the chosen data.
            printfiles = boolean (default = False)
                - If False, the filenames associated with the data retrieved
                    ARE NOT printed.
                - If True, the filenames associated with the data retrieved ARE
                    printed.
            gridded_int = boolean (default = True)
                - If True, the variables chosen are functionalized in both the
                    standard method and a gridded method.
                - If False, the variables chosen are functionalized in only the
                    standard method.
            verbose = boolean (False)
                - If False, script execution and the underlying Kamodo
                    execution is quiet except for specified messages.
                - If True, be prepared for a plethora of messages.

        Returns: a kamodo object (see Kamodo core documentation) containing all
            requested variables in functionalized form.

        Notes:
            - This model reader is the most basic example of what is required
              in a model reader. The only 'data wrangling' needed is a simple
              longitude wrapping and numpy array transposition to get the
              coordinate order correct.
            - DTM outputs files are given in one netCDF file per day.
            - The files are small and contain multiple time steps per file, so
              interpolation method 2 is chosen. The standard SciPy interpolator
              is used.
        '''

        def __init__(self, file_dir, variables_requested=[],
                     filetime=False, verbose=False, gridded_int=True,
                     printfiles=False, **kwargs):
            super(MODEL, self).__init__()
            self.modelname = 'DTM'
```
This preamble is generally copied between readers and modified in the new reader script to reflect the varying import statements and the model-specific content in the Notes section. Otherwise, no changes to the preamble are made between readers. The last line simply establishes the name of the model associated with the model outputs supported by the model reader, which should match the list of model reader names at the top of the model_wrapper.py script (in the /Kamodo/kamodo_ccmc/flythrough/ directory). 

## Time and List Files Contents
The next section of code in the model reader creates two files that contain the file and timing information for the run stored in the given file directory (file_dir). Before considering the code, it is beneficial to discuss the outputs. The DTM_list.txt file contains a list of the data files in the given directory, followed by the start and end dates and times in UTC.

Sample file contents:  
DTM file list start and end dates and times  
D:/DTM/Joshua_Anumolu_101722_IT_3/DTM.2013151.nc  Date: 2013-05-31  Time: 00:00:00    Date: 2013-05-31  Time: 23:45:00  
D:/DTM/Joshua_Anumolu_101722_IT_3/DTM.2013152.nc  Date: 2013-06-01  Time: 00:00:00    Date: 2013-06-01  Time: 23:45:00  
D:/DTM/Joshua_Anumolu_101722_IT_3/DTM.2013153.nc  Date: 2013-06-02  Time: 00:00:00    Date: 2013-06-02  Time: 23:45:00  
D:/DTM/Joshua_Anumolu_101722_IT_3/DTM.2013154.nc  Date: 2013-06-03  Time: 00:00:00    Date: 2013-06-03  Time: 23:45:00  

The DTM_times.txt file contains the full array of time values in the data for each file in the file directory. The time values are given in HH:MM:SS since midnight of the first day of data represented in the data files. In this case, the first file starts on May 31, 2013 and the last file ends on June 3, 2013, covering four days or approximately 96 hours as indicated by the time grid values in the DTM_times.txt file. Other model outputs are given in multiple sets of files covering the same time range (e.g. CTIPe and WACCM-X), and often with differing time cadences. In those cases, the time grid values are given separately for each set of files (e.g. the height, neutral, and density file sets of CTIPe data).

Sample file contents (shortened for brevity):  
DTM time grid per pattern  
Pattern: DTM  
00:00:00  
00:15:00  
00:30:00  
00:45:00  
01:00:00  
01:15:00  
01:30:00  
...  
94:45:00  
95:00:00  
95:15:00  
95:30:00  
95:45:00  

The contents of the files above are produced by the code from the DTM reader below, which will be explained piece by piece. The logic required to produce these outputs must collect the names of all the files, sorted and categorized by naming pattern, and the various timing information from each file.

## Creating the Time and List Files
The chunk of code described in this section checks for the presence of the list and time files just discussed, and then creates them if the files are not found. The if statement below is the standard place to put calls to any necessary file conversion or pre-processing routines. As a rule, these routines should be defined in a script external to the model reader, but named with the same model name to show association (e.g. gitm_tocdf.py for GITM's file conversion routines). The logic in the model reader script that calls these external functions should be designed in a way to only execute if the time and list files are not found, only execute if the resulting files are not found (e.g. the \*.nc files for the GITM output), and return the MODEL class object as is if the routine fails to complete successfully.  

The decision to include a file conversion routine is important. The general rule of thumb is that if you need more than a few lines of code to access the data stored in the file, then a file conversion routine is likely needed. More importantly, if it takes longer to read in the data from the current file format than from a cdf or h5 file, then the file conversion step should be developed. Alternatively, if the time taken to perform the required data manipulations and processing steps is deemed significant, then a file conversion step should be added to perform these steps the first time the reader is run rather than during the interpolation calls.

One goal of the code below is to avoid repeatedly searching the given file directory for data files, particularly avoiding repeated glob calls. This choice is made to make Kamodo cheaper and faster to run in a cloud environment where data is stored in s3 buckets. However, this choice is  problematic for running the readers on top of data being produced in real-time. More research is required to determine what modifications are needed to adapt these readers for real-time applications.

The first section of code below simply establishes the standard names of the list and times files and create the times and pattern_files dictionary class attributes.
```py
            # first, check for file list, create if DNE
            list_file = file_dir + self.modelname + '_list.txt'
            time_file = file_dir + self.modelname + '_times.txt'
            self.times, self.pattern_files = {}, {}
            if not isfile(list_file) or not isfile(time_file):
```
If both files are not detected in the given file directory, then they are created by the remainder of the code in this section.
```py
                t0 = perf_counter()  # begin timer
                # figure out types of files present
                files = sorted(glob(file_dir+'*.nc'))
                patterns = sorted(unique([basename(f)[:-11] for f in
                                          files]))  # cut off date
```
After the timer is set, the glob statement retrieves a list of the data files in the given file directory. In this case, the listing logic is simple because all of the data are given in cdf files. In other cases, the data is given in txt files and some filtering is needed to remove the list and time files from this list (and any other irrelevant files in the directory). The patterns line is written to remove the time or processor specific information from the file names and determine what the naming patterns of the files are. For example, the file names of the DTM files are  
DTM.2013151.nc  
DTM.2013152.nc  
DTM.2013153.nc  
DTM.2013154.nc  
where the numerals indicate the year and day of year for the data. The filtering logic for DTM is written to assume that the characters preceding the timing information can be of varying length, so the filtering is done by chopping off the ending characters instead of saving the beginning characters. In the DTM case, only one naming pattern is known. Other model outputs have multiple naming patterns, which can be of varying length, but all with the same number of characters indicating the timing or processor information. The convention taken across the model readers is to chop of the characters indicating the timing or processor information from the end of the string closest to those characters and treat the remaining characters as the naming convention(s). In this case, the timing information is at the end of the file names, so the positions are always determined from the end of the filename.

```py
                self.filename = ''.join([f+',' for f in files])[:-1]
                self.filedate = datetime.strptime(
                    basename(files[0])[-10:-3]+' 00:00:00', '%Y%j %H:%M:%S'
                    ).replace(tzinfo=timezone.utc)
```
The names of all the files stored in the files variable are combined into a list stored in the filename class attribute, which will be part of the output of the DTM_list.txt file as shown earlier. The next line of code calculates the datetime object for midnight in UTC of the first day of the time range in the data. In the DTM case, the first date can be determined from the filename of the first file in the sorted file list in the files variable. In other cases, this must be determined by opening the file and retrieving the information directly.

The values of the time grid may be different for different sets of files (e.g. WACCM-X). So, the values must be retrieved and stored for each naming pattern. The code below loops through the different naming patterns stored in the patterns variable. First, a sorted list of the files corresponding to the given naming pattern are retrieved and stored in the pattern_files dictionary attribute. A new dictionary is then created in the times dictionary attribute to store the start time, end time, and the full array of time values for the given file naming pattern. This results in a nested dictionary. The choice of lists for the three dictionary values is especially useful since there is no way to know beforehand how many time values are in each file, and array concatenation is expensive.
```py
                # establish time attributes
                for p in patterns:  # only one pattern
                    # get list of files to loop through later
                    pattern_files = sorted(glob(file_dir+p+'*.nc'))
                    self.pattern_files[p] = pattern_files
                    self.times[p] = {'start': [], 'end': [], 'all': []}
```

In cases where the data are stored with one time step per file, the time values can usually be retrieved from the filenames. Then, the lists in the start, end, and all dictionary keys are identical to each other. For the DTM data, multiple time steps are stored in each file, so the each file must be opened to retrieve the values, and the values for the start, end and all dictionary keys will be different.
```py
                    # loop through to get times, one day per file
                    for f in range(len(pattern_files)):
                        cdf_data = Dataset(pattern_files[f])
                        # minutes since 12am EACH file -> hrs since 12am 1st f
                        tmp = array(cdf_data.variables['time'])/60. + f*24.
```
The tmp array defined on the last line of code above must be navigated with care. The values stored in the times dictionary must be in hours since midnight of the day of the first file. The DTM time values are stored in minutes since midnight of each day, so they must be converted to hours and adjusted for each file (e.g. add 24 times the file number to the values). This logic changes between model readers depending on the units and conventions of the times stored in the data.  
The first time value from each file is stored in the start list, the last value of each file is stored in the end list, and the entire array of values from each file is stored in the all list (see code below). The logic is repeated for each file, and then the three lists are converted into NumPy arrays.
```py                        
                        self.times[p]['start'].append(tmp[0])
                        self.times[p]['end'].append(tmp[-1])
                        self.times[p]['all'].extend(tmp)
                        cdf_data.close()
                    self.times[p]['start'] = array(self.times[p]['start'])
                    self.times[p]['end'] = array(self.times[p]['end'])
                    self.times[p]['all'] = array(self.times[p]['all'])
```
These collected values are then passed on to a standard routine called *create_timelist* in the reader_utilities.py script (in the Kamodo/kamodo_ccmc/readers/ directory) to create the time and list files.
```py
                # create time list file if DNE
                RU.create_timelist(list_file, time_file, self.modelname,
                                   self.times, self.pattern_files,
                                   self.filedate)

            else:  # read in data and time grids from file list
                self.times, self.pattern_files, self.filedate, self.filename =\
                    RU.read_timelist(time_file, list_file)
```
If the time and list files already exist, then all of this logic is skipped. Instead, the times, pattern_files, filedate and filename variables are retrieved from the two files via a standard routine called *read_timelist* in the reader_utilities.py script (in the Kamodo/kamodo_ccmc/readers/ directory).
```py
            if filetime:
                return  # return times as is 
```
If the filetime keyword is set to True, then the script returns. The requested time values are stored in the times attribute of the returned Kamodo object. This behavior is used by the *File_Times* function in the model_wrapper script (in the Kamodo/kamodo_ccmc/flythrough/ directory), which returns the time information to the user.

## Checking the Variables Requested
There are several blocks of code that address a variety of scenarios concerning the variables_requested variable (the list of strings corresponding to the LaTeX representation of the variable names). Before diving into these, a few lines of code typically occur at this point in the model reader to initialize the data structures needed later.
- The missing_value attribute is typically defined as a NaN (imported from numpy), but should change depending on the convention taken in the data files. The given value is replaced with a NaN before the interpolator is defined.  
- The varfiles dictionary is used to store what variable names (LaTeX representation) are found in what category of files (e.g. the height files vs the neutral files for the CTIPe data). The keys of this dictionary are set to be the keys of the pattern_files dictionary attribute defined earlier, and the values are a list of the requested variable names found in the files of the same naming pattern as the key.
- The gvarfiles dictionary is identical in structure, except the values are a list of the variable names as given in the file for each naming pattern. Recall that the variable names as given in the file are the keys of the model_varnames dictionary at the top of each model reader script, and the LaTeX representation of each variable is always the first element in the list associated with the given key.  
- The err_list attribute is simply a list of the variable names (LaTeX representation) requested by the user but not found in the data files.  
```py
            # store variables
            self.missing_value = NaN
            self.varfiles = {}  # store which variable came from which file
            self.gvarfiles = {}  # store file variable name similarly
            self.err_list = []
```  

The first check on the items in the variables_requested list is to ensure that the variable names requested are in the model_varnames dictionary. If one or more variable names are found that are not in the model_varnames dictionary, then those strings are removed from the list and a message is printed with a list of the incorrect variable names ('Variable name(s) not recognized...'). If all of the requested variable names are not recognized, the MODEL class instance is returned as is.  
```py
            # perform initial check on variables_requested list
            if len(variables_requested) > 0 and variables_requested != 'all':
                test_list = [value[0] for key, value in model_varnames.items()]
                err_list = [item for item in variables_requested if item not in
                            test_list]
                if len(err_list) > 0:
                    print('Variable name(s) not recognized:', err_list)
                for item in err_list:
                    variables_requested.remove(item)
                if len(variables_requested) == 0:
                    return
```             
In several model readers, the remainder of the variable checks in the next chunk are performed in a loop with the file naming patterns as the iterating variable. Since the coordinate grids are often different between these file sets, they are also retrieved in the same loop. (See the CTIPe and the GITM model readers for an example). For DTM, there is only one naming pattern, so this loop is removed by storing the single naming pattern in the variable p and the associated files in the variable pattern_files (instead of looping over the different values for p).

At this point in the reader script, the data file must be accessed to retrieve a list of the variables and coordinate data stored in the file. Once the data file is opened, the expected variable names in the file for the requested variables are compared with the variable names found in the file. In other words, the keys of the model_varnames dictionary corresponding to the LaTeX representation of the requested variables are compared with the variable names found in the file. The names of the matched variables as given in the files are saved in the gvar_list variable, and the corresponding LaTeX representation of those variables are saved in the varfiles attribute. In readers requiring the loop just discussed, the list of matches is stored in self.gvar_list\[p\] instead of the simpler list structure below. The LaTeX representation of the requested variables that are not found in the data files are stored in the err_list attribute, which is used for error printing later in the script.

If the reader is executed without a list of requested variables (i.e. if the variables_requested list is left empty), then the logic assumes that the user wishes to functionalize all of the variables found in the data. In this case, no matching is needed and the full list of variable names as represented in the file and included in the model_varnames dictionary is stored in the gvar_list variable and later also in the gvarfiles attribute with the associated file naming pattern as the key. Similarly, a list of the LaTeX representation of those variables is stored in the varfiles attribute with the same key.
```py
            # there is only one pattern for DTM, so just save the one grid
            p = list(self.pattern_files.keys())[0]
            pattern_files = self.pattern_files[p]
            cdf_data = Dataset(pattern_files[0], 'r')

            # check var_list for variables not possible in this file set
            if len(variables_requested) > 0 and\
                    variables_requested != 'all':
                gvar_list = [key for key in model_varnames.keys()
                             if key in cdf_data.variables.keys() and
                             model_varnames[key][0] in variables_requested]
                if len(gvar_list) != len(variables_requested):
                    err_list = [value[0] for key, value in
                                model_varnames.items()
                                if key not in cdf_data.variables.keys() and
                                value[0] in variables_requested]
                    self.err_list.extend(err_list)  # add to master list
            else:
                gvar_list = [key for key in model_varnames.keys()
                             if key in cdf_data.variables.keys()]

            # store which file these variables came from
            self.varfiles[p] = [model_varnames[key][0] for
                                key in gvar_list]
            self.gvarfiles[p] = gvar_list

            # get coordinate grids from first file
            self._lat = array(cdf_data.variables['lat'])  # -90 to 90
            lon = array(cdf_data.variables['lon'])  # 0 to 360
            lon_le180 = list(where(lon <= 180)[0])  # 0 to 180
            lon_ge180 = list(where((lon >= 180) & (lon < 360.))[0])
            self._lon_idx = lon_ge180 + lon_le180
            self._lon = lon - 180.
            self._height = array(cdf_data.variables['ht'])  # km
            cdf_data.close()
```  
The bottom block of code retrieves the coordinate grid for the given file naming pattern. The lon_le180, lon_ge180, and \_lon_idx variables are used to change the longitude range from \[0, 360\] to \[-180, 180\] without changing the position that the zero longitude refers to in the data (see also the last line of the *func* function defined later in the script). Note that the time coordinate values are retrieved earlier in the script. In more complex situations, the coordinate grids are stored as class attributes named with the coordinate name combined with the associated file naming pattern (e.g. the CTIPe and GITM model readers).

The remaining two blocks of code conclude the variable checks typically performed in a model reader. The first simply prints an error message indicating what requested variables were not found in the data files. The second block (if variables_requested == 'all') creates a slightly reduced version of the model_varnames dictionary and returns it as a class attribute. This is required by various functions in the model_wrapper.py script (in the Kamodo/kamodo_ccmc/flythrough/ directory) that enable the variable search features demonstrated in the Choosing Models and Variables section of the documentation.
```py
            # print message if variables not found
            if len(self.err_list) > 0:
                print('Some requested variables are not available: ',
                      self.err_list)

            # collect all possible variables in set of files and return
            if variables_requested == 'all':
                var_list = self.varfiles[p]
                self.var_dict = {value[0]: value[1:] for key, value in
                                 model_varnames.items() if value[0] in
                                 var_list}
                return
```               
In some cases, more logic is needed to fully navigate the mapping between the user requested variables and the data files, particularly when two versions of a given variable are required. The most drastic example of this occurs in the WACCM-X model reader script, and a simpler example is found in the WAM-IPE script.

Before functionalizing the requested variable data, the printfiles keyword is checked. If True, then the names of all of the data files found in the given file directory are printed to the screen - one file per line for clarity.
```py
            # option to print files
            if printfiles:
                print(f'{len(self.filename)} Files:')
                files = self.filename.split(',')
                for f in files:
                    print(f)
```

## Storing the Variable Mapping
The mapping between the variable names and the data files is stored in the first block of code below in the variables dictionary. The keys of the dictionary are the LaTeX representations of the variables requested and found in the data. The value for each key is itself a dictionary, with the standard strings 'units' and 'data' as the keys. The string giving the units for each variable is stored as the value for the 'units' key, and the file naming pattern corresponding to the files in which the variable data are found is stored as the value for the 'data' key.  

In previous Kamodo versions, the 'data' portion of the variables dictionary was used to store the actual data found in the file(s) for the corresponding variable. However, that approach was abandoned when the decision was made to functionalize the entire dataset found in a given directory instead of a single file or subset of files simply because it cannot be assumed that the dataset can fit into memory.

```py
            # initialize storage structure
            self.variables = {model_varnames[gvar][0]: {
                'units': model_varnames[gvar][-1], 'data': p} for gvar in
                self.gvarfiles[p]}

            # register interpolators for each variable
            t_reg = perf_counter()
            # store original list b/c gridded interpolators change keys list
            varname_list = [key for key in self.variables.keys()]
            for varname in varname_list:
                self.register_variable(varname, gridded_int)

            if verbose:
                print(f'Took {perf_counter()-t_reg:.5f}s to register ' +
                      f'{len(varname_list)} variables.')
            if verbose:
                print(f'Took a total of {perf_counter()-t0:.5f}s to kamodofy' +
                      f' {len(varname_list)} variables.')
```
The second block of code marks the time, stores a list of the keys of the variables dictionary in the varname_list variable, and loops through the list of variables, calling the register variable function for each one. The creation of the varname_list variable is required because the variables dictionary is changed in the functionalization process, which causes errors in the looping process if the keys of that dictionary are used as the looping variable. Recall that the keys of the variables dictionary are the LaTeX representation of the variables surviving the checks above. The remaining two blocks of code simply print timing messages if the verbose keyword is True. The first prints the number of seconds taken to loop through the variable names in the preceding block, while the second prints the total time taken for the model reader to execute.                       

## Functionalizing the Data
### The register_variable function
The register_variable function begins below and creates an interpolator for each variable in a sequence of steps. 
- The file naming pattern associated with the variable is retrieved and stored in the key variable, 
- The name of the variable in the file is saved in the gvar variable,
- A list of the coordinate names that the variable depends on is extracted from the model_varnames dictionary. This coordinate list is used a few lines down to determine the dimensionality of the variable data (if len(coord_list) == 4: ...). 
- A dictionary of the coordinate grids is constructed in the same order as the coordinate name list: time first, followed by longitude and latitude, and ending with height if the variable depends on height. The coordinate names are used as the keys for the coord_dict dictionary, with values that are themselves dictionaries (resulting in a nested dictionary). Just as with the variables dictionary, the strings 'units' and 'data' are used here to store the unit associated with the given coordinate (e.g. 'hr' for time) and the values for the coordinate grid. The values for the time grid are taken from the times dictionary defined at the beginning of the model reader script. The values for the remaining coordinates are taken from the data stored in the class attribute. 
- Finally, the two strings indicating the coordinate system are combined into a single string stored in the coord_str variable (e.g. GDZsph). This string is later added as a subscript to the LaTeX representation of the coordinates (see the image below).
![Screenshot](Files/SampleLaTeXFunction.png)
```py
        def register_variable(self, varname, gridded_int):
            """Registers an interpolator with proper signature"""

            # determine which file the variable came from, retrieve the coords
            key = self.variables[varname]['data']
            gvar = [key for key, value in model_varnames.items() if
                    value[0] == varname][0]  # variable name in file
            coord_list = [value[-2] for key, value in
                          model_varnames.items() if value[0] == varname][0]
            coord_dict = {'time': {'units': 'hr',
                                   'data': self.times[key]['all']}}
            # get the correct coordinates
            coord_dict['lon'] = {'data': self._lon, 'units': 'deg'}
            coord_dict['lat'] = {'data': self._lat, 'units': 'deg'}
            if len(coord_list) == 4:
                coord_dict['height'] = {'data': self._height, 'units': 'km'}
            coord_str = [value[3]+value[4] for key, value in
                         model_varnames.items() if value[0] == varname][0]
```  
In cases where more than one file naming pattern is present, the coordinate grids associated with those files are assumed to be different. The logic below already accounts for possibly differing time grids, but not for the other coordinate grids. In those cases, the key variable is incorporated into the logic that retrieves the coordinate grid. Consider the excerpt below from the GITM script:
```py
            coord_dict = {'time': {'units': 'hr',
                                   'data': self.times[key]['all']}}
            coord_dict['lon'] = {
                'data': getattr(self, '_lon_'+key), 'units': 'deg'}
            coord_dict['lat'] = {
                'data': getattr(self, '_lat_'+key), 'units': 'deg'}
            if len(coord_list) == 4:
                coord_dict['height'] = {
                    'data': getattr(self, '_height_'+key),
                    'units': 'km'}
```
Note the time grid is retrieved with code identical to the DTM example above, but the key variable is required to retrieve the coordinate grids.

### Choosing the Interpolation Method
Returning to the DTM model reader script, the next task is to functionalize the data associated with the requested variable. This requires reading in the data, performing any data manipulations needed, and constructing an interpolator to interpolate through the data both through time and through all spatial dimensions. The decision to interpolate/functionalize the entire dataset found in the given directory precludes the option to read in all of the data to enable interpolation through the entire dataset at once. So, a code structure was created to "lazily" interpolate through time by loading only the time steps or chunks needed to perform a given interpolation. The function handling the lazy interpolation through time is the *Functionalize_Dataset* function in the reader_utilities.py script (in the Kamodo/kamodo_ccmc/readers/ directory). The code above constructs all of the necessary inputs to this function except the function needed and the interpolation method appropriate for the data. The function documentation is below. A full description of this logic is available in the Kamodo model reader paper (DOI HERE WHEN AVAILABLE).    


```python
from kamodo_ccmc.readers.reader_utilities import Functionalize_Dataset
help(Functionalize_Dataset)
```

    Help on function Functionalize_Dataset in module kamodo_ccmc.readers.reader_utilities:
    
    Functionalize_Dataset(kamodo_object, coord_dict, variable_name, data_dict, gridded_int, coord_str, interp_flag=0, func=None, times_dict=None, func_default='data')
        Determine and call the correct functionalize routine.
        Inputs:
            kamodo_object: the previously created kamodo object.
            coord_dict: a dictionary containing the coordinate information.
                {'name_of_coord1': {'units': 'coord1_units', 'data': coord1_data},
                 'name_of_coord2': {'units': 'coord2_units', 'data': coord2_data},
                 etc...}
                coordX_data should be a 1D array. All others should be strings.
            variable_name: a string giving the LaTeX representation of the variable
            data_dict: a dictionary containing the data information.
                {'units': 'data_units', 'data': data_array}
                If interp_flag=0 is used, data_array should be a numpy array of
                shape (c1, c2, c3, ..., cN). Otherwise, it could be a string or
                other thing that feeds into the given func.
            Note: The dataset must depend upon ALL of the coordinate arrays given.
        
            gridded_int: True to create a gridded interpolator (necessary for
                plotting and slicing). False otherwise (e.g. for the flythrough).
            coord_str: a string indicating the coordinate system of the data
                (e.g. "SMcar" or "GEOsph").
            interp_flag: the chosen method of interpolation.
                Options:
                    0: (default option) assumes the given data_dict['data'] is an
                        N-dimensional numpy array and creates a standard scipy
                        interpolator to functionalize the data. This option should
                        be used when the entire dataset is contained in a single
                        array AND can easily fit into memory (e.g. 1D time series)
                    1: Lazy interpolation is applied on top of the standard scipy
                        interpolators (one per time slice). This option is designed
                        to be used when the data is stored in one time step per
                        file and the time step can easily fit into memory.
                    2: Lazy chunked interpolation is applied on top of the
                        standard scipy interpolators (one per time chunk).
                        Interpolation between time chunks occurs in the given
                        func. This option is designed to be used when the data is
                        stored with more than one time step per file AND the time
                        chunks are small enough to easily fit into memory.
                    3. Combination of 1 and 2, used when the time chunks are too
                        large for the typical computer memory. Searches for the
                        correct time chunk, then the correct time slice in the
                        time chunk. This option is designed to be used when the
                        files contain more than one time step and are too large to
                        easily read into memory.
            func: a function defining the logic to be executed on a given time
                slice or chunk (e.g. converting to an array and then transposing
                it).
                *** Only needed for interp_flag greater than zero. ***
                - For interp_flag=1, the function should return only the data for
                    the time slice. Required syntax is data = func(i), where i is
                    the index of the slice on the full time grid.
                - For interp_flag=2, the function should return the data for the
                    time chunk. Required syntax is data = func(i), where i is the
                    file number in a list of files of the same naming pattern.
                - For interp_flag=3, the function should return only the data for
                    the time slice. Required syntax is data = func(i, fi), where i
                    is the file number in a list of files of the same naming
                    pattern, and fi is the index of the time slice in that file.
                - ALL functions must include the method to retrieve the data from
                    the file.
                - This option is useful when a dataset's shape order does not match
                the required order (e.g. t, lat, lon -> t, lon, lat) and allows
                such operations to be done on the fly.
            start_times: a list of the start times for each data chunk.
                *** Only needed for interpolation over time chunks. ***
                (interp_flag options 2 and 3).
            func_default: a string indicating the type of object returned by func.
                The default is 'data', indicating that the returned object is a
                numpy array. Set this to 'custom' to indicate that func returns
                a custom interpolator.
        
        Output: A kamodo object with the functionalized dataset added.
    
    

The interpolation method should be chosen based on the data properties, specifically the dimensionality of the variable data, the number of time steps stored in each file, and the typical size of the data files. Variable data depending only on time or only on spatial coordinates best fit interpolation method 0. Time-varying data of higher dimensions should use one of the other interpolation methods. If the data is stored with one time step per file, then interpolation method 1 is the best choice regardless of the file size. If the data is stored with multiple time steps per file ("chunks") and the file sizes are a few GB or less, then interpolation method 2 should be used. If the data is stored in chunks with file sizes larger than a few GB, then interpolation method 3 is recommended to avoid memory errors.  

Interpolation method 0 assumes that the variable data fits in memory, and so does not support lazy interpolation in time. The remainder of the interpolation methods are used for variable data that do not fit in memory and offer different versions of the lazy interpolation in time logic based on the chosen method. These interpolation methods therefore require that the script developer define a function to be executed for each portion of the data. This script must read the portion of data into memory, perform any processing required, and return the data in the standard form described earlier (e.g. dimension 0=time, 1=longitude, etc).  

For data associated with coordinate grids that do not change with time or position, the interpolation is handled through a standard interface through the *RU.Functionalize_Dataset* function to the SciPy *RegularGridInterpolator* interpolating function for data of multiple dimensions and through a similar standard interface to the SciPy *interp1d* interpolating function. (See SciPy's documentation on these at https://docs.scipy.org/doc/scipy/reference/generated/scipy.interpolate.RegularGridInterpolator.html and https://docs.scipy.org/doc/scipy/reference/generated/scipy.interpolate.interp1d.html#scipy.interpolate.interp1d). For datasets that do not fall in this category, a custom interpolator is required (see related sectio below). In the DTM data, the data has two to three spatial dimensions, and the coordinate grids do not vary with time or in the spatial dimensions, so the standard interface to the SciPy *RegularGridInterpolator* is sufficient.

Once the function is defined and the call to the *RU.Functionalize_Dataset* is included, the model reader should be concluded with a return statement for the MODEL function that returns the MODEL class.  
```py
return MODEL
```

#### Interpolation Method 0
Interpolation method 0 is used for one-dimensional datasets (e.g. only dependent on time) and for variables that are constant in time. The entire dataset for the given variable is typically read into memory and functionalized in the standard way. The code below gives an example of this from the superdarnuni_4D.py reader.
```py
            # retrieve coordinate grids
            if len(coord_list) == 1:  # read in entire time series
                data = []
                for f in self.pattern_files[key]:
                    cdf_data = Dataset(f)
                    tmp = array(cdf_data.variables[gvar])[0]
                    cdf_data.close()
                    data.append(tmp)  # one value per file
                self.variables[varname]['data'] = array(data)
                self = RU.Functionalize_Dataset(
                    self, coord_dict, varname, self.variables[varname],
                    gridded_int, coord_str, interp_flag=0)  # single array
                return
```
If the variable depends on a single dimension, in this case only time, then the data for that variable are collected from the entire dataset (cdf files in this case), stored in the self.variables dictionary in the data key, and functionalized with the call to the *RU.Functionalize_Dataset* standard interface. Note the *interp_flag* keyword is set to zero in this call and no function is included in the call. This is because the lazy interpolation supported by the other interpolation method values is not needed here. If a custom interpolator is required for this method, then a function must be defined that returns the interpolator.

#### Interpolation Method 1
Interpolation method 1 is used for datasets where the data is stored with one time step per file and depends on spatial coordinates. This method supports lazy interpolation in time, which requires that a function be defined as described earlier. In this case, a simple version of the required code is found in the GITM model reader (Kamodo/kamodo_ccmc/readers/gitm_4Dcdf.py) and included below. The logic defines the function to be executed on each time slice of the data required for the given interpolation. 
```py
            def func(i):
                '''i is the file number.'''
                # get data from file
                file = self.pattern_files[key][i]
                cdf_data = Dataset(file)
                data = array(cdf_data.variables[gvar])
                cdf_data.close()
                return data

            # define and register the interpolators
            self = RU.Functionalize_Dataset(self, coord_dict, varname,
                                            self.variables[varname],
                                            gridded_int, coord_str,
                                            interp_flag=1, func=func)                
```                
In this case, all data processing has been done in a file conversion step, so the code simply reads in the required time step from the correct file and returns the data for that time step. Note that the only argument for the *func* function is an integer, *i*, which is the position of the given timestep in the dataset and the file number in the self.pattern_files list. In comparison to the 0th interpolation method, the call to *RU.Functionalize_Dataset* now includes the function defined.

#### Interpolation method 2
Interpolation method 2 is used for datasets where data is stored with multiple time steps per file and the file sizes are small, typically a few GB or less. This method supports lazy interpolation in time, which requires that a function be defined as described earlier. In this case, a simply version of the required code is found in our example reader, the DTM reader. The logic defines the function to be executed on each time "chunk" of the data required for a given interpolation.  
```py
            # define operations for each variable when given the key
            def func(i):
                '''key is the file pattern, start_idxs is a list of one or two
                indices matching the file start times in self.start_times[key].
                '''
                # get data from file
                file = self.pattern_files[key][i]
                cdf_data = Dataset(file)
                data = array(cdf_data.variables[gvar])
                if hasattr(cdf_data.variables[gvar][0], 'fill_value'):
                    fill_value = cdf_data.variables[gvar][0].fill_value
                else:
                    fill_value = None
                cdf_data.close()
                # if not the last file, tack on first time from next file
                if file != self.pattern_files[key][-1]:  # interp btwn files
                    next_file = self.pattern_files[key][i+1]
                    cdf_data = Dataset(next_file)
                    data_slice = array(cdf_data.variables[gvar][0])
                    cdf_data.close()
                    data = append(data, [data_slice], axis=0)
                # data wrangling
                if fill_value is not None:  # if defined, replace with NaN
                    data = where(data != fill_value, data, NaN)
                if len(data.shape) == 3:
                    variable = transpose(data, (0, 2, 1))
                elif len(data.shape) == 4:
                    variable = transpose(data, (0, 3, 2, 1))
                return variable[:, self._lon_idx]

            self = RU.Functionalize_Dataset(
                self, coord_dict, varname, self.variables[varname],
                gridded_int, coord_str, interp_flag=2, func=func,
                times_dict=self.times[key])
```
The code required for interpolation method 2 is more complex than the other interpolation methods because it must not only load the data for the chunk required, but it must also load the data for the first time step in the next chunk to enable proper interpolation in time between files. This example also includes logic to test for the presence of a fill value attribute in the data. The block labeled "data wrangling" replaces the given fill value, if defined, with NaN values (imported from NumPy), transposes the array to match the proper coordinate order, and handles the longitude coordinate shift in the return statement. The return statement returns the prepared data stored in the variable array as required by the logic in the *Functionalize_Dataset* function.

The only argument accepted by the defined function is an integer *i*, which represents the position of the data chunk in the full dataset, which is the same as the file number in the self.pattern_files list.  Note that the call to the *RU.Functionalize_Dataset* includes the function defined as in the interpolation method 1 example, and also includes the times dictionary. 

#### Interpolation Method 3
Interpolation method 3 is used for datasets where data is stored with multiple time steps per file and the file sizes are larger than a few GB. This method supports lazy interpolation in time, which requires that a function be defined as described earlier. In this case, a simply version of the required code is found in the model reader script for WACCM-X. Instead of working with the entire chunk at once, the interpolation logic works with the data for one time step at a time to avoid memory errors. So, the logic defined in the *func* function below is executed on each time slice of the data required for a given interpolation. 
```py
            def func(i, fi):  # i = file#, fi = slice#
                cdf_data = Dataset(self.pattern_files[key][i])
                tmp = array(cdf_data.variables[gvar][fi])
                cdf_data.close()
                # perform data wrangling on time slice
                # (ilev,) lat/mlat, lon/mlon -> lon/mlon, lat/mlat, (ilev)
                data = tmp.T[lon_idx]
                return data

            # functionalize the 3D or 4D dataset, series of time slices
            self = RU.Functionalize_Dataset(
                self, coord_dict, varname, self.variables[varname],
                gridded_int, coord_str, interp_flag=3, func=func,
                times_dict=self.times[key])
```                
Note that the function defined for this interpolation method requires two input arguments as compared to the functions defined for the previous interpolation methods which only have one input argument. The arguments required by the *func* function defined above are i, an integer representing the position of the time chunk and file in the dataset, and fi, an integer representing the position of the time step in the given time chunk. As shown in the interplation method 1 example, the data for the given time slice is retrieved from the file, some simple processing is performed, and the data is returned as required. The keywords included in the call to *RU.Functionalize_Dataset* are identical to those shown in interpolation method 2, except the interp_flag keyword is set to 3.

#### Custom interpolators
All of the interpolation methods shown above support the use of custom interpolators. If a custom interpolator is desired, the script developer should define the logic for that interpolator in a separate script named with the model name and 'interp' or some other string indicating it is an interpolation routine. The routine should be returned by the *func* function and included in the call to *RU.Functionalize_Dataset*, and the *func_default* keyword should be set to 'custom'. A simple example can be found in the superdarnequ_4D script in the Kamodo/kamodo_ccmc/readers/ directory.
```py
            from kamodo_ccmc.readers.superdarnequ_interp import custom_interp

            def func(i):  # i is the file/time number
                # get data from file(s)
                cdf_data = Dataset(self.pattern_files[key][i])
                lat = array(cdf_data.variables['lat'])
                lat_keys = [str(lat_val).replace('.', '_').replace('-', 'n')
                            if lat_val < 0 else 'p'+str(lat_val).replace(
                                    '.', '_') for lat_val in lat]
                lon_dict = {lat_val: array(cdf_data.groups[lat_key]['lon'])
                            for lat_key, lat_val in zip(lat_keys, lat)}
                data = {lat_val: array(cdf_data.groups[lat_key][gvar])
                        for lat_key, lat_val in zip(lat_keys, lat)}
                cdf_data.close()

                # assign custom interpolator
                interp = custom_interp(lon_dict, lat, data)
                return interp

            # functionalize the variable dataset
            tmp = self.variables[varname]
            tmp['data'] = zeros((2, 2, 2))  # saves execution time
            self = RU.Functionalize_Dataset(
                self, coord_dict, varname, tmp, gridded_int, coord_str,
                interp_flag=1, func=func, func_default='custom')
```
The section of code begins with importing the custom interpolator script, which is then initialized with the data required for a given time step. (The superdarnequ_4D.py model reader script uses interpolation method 1.) A few extra steps are performed to prevent automatic execution of the interpolator, namely setting the data attribute of the self.variables dictionary to an array of zeros with the same dimensionality of the data for the time step (in this case, three dimensions for the longitude, latitude, and height dimensions in the data). A more complex example of a custom interpolator is given in the swmfgm_4D.py script (in the Kamodo/kamodo_ccmc/readers/ directory).

## Model Reader Testing
Once the developer has an initial version of the model reader script, it should be tested in a variety of ways. See the /Kamodo/Validation/Notebooks/ directory for the notebooks used to test each model reader for a full array of functionalities. These tests include generation of the list and times files, proper variable handling, proper behavior of the interpolators for one variable of each dimensionality, correct plot creation for the same variables, proper creation of the model_varnames metadata dictionary through the metadata search functions, and proper integration into the flythrough functions. Each model reader must successfully pass these tests and any others needed to verify full functionality before it can be included in the Kamodo software. 

Before running these tests on the newly developed model reader, the developer should include the model reader script into the model_wrapper.py script located in the Kamodo/kamodo_ccmc/flythrough directory through the following two additions:
- The model information should be added to the model_dict dictionary at the top of the model_warpper.py script in alphabetical order. The acronym representation of the model should be added as a key in the dictionary with the full name and associated DOI(s) as the value associated with the key, as shown below, and
```py
model_dict = {'ADELPHI': 'AMPERE-Derived ELectrodynamic Properties of the ' +
                         'High-latitude Ionosphere ' +
                         'https://doi.org/10.1029/2020SW002677',
              'AMGeO': 'Assimilative Mapping of Geospace Observations ' +
                       'https://doi.org/10.5281/zenodo.3564914',
              'CTIPe': 'Coupled Thermosphere Ionosphere Plasmasphere ' +
                       'Electrodynamics Model ' +
                       'https://doi.org/10.1029/2007SW000364',
              'DTM': 'The Drag Temperature Model ' +
                     'https://doi.org/10.1051/swsc/2015001',
```  
- An elif logic block should be added in the *Choose_Model* function before the else statement. The example below shows the logic included for the DTM model reader script.
```py
    elif model == 'DTM':
        import kamodo_ccmc.readers.dtm_4D as module
        return module
```  

Finally, the model reader script should align with PEP8 standards. This can be tested by running the following commands in a terminal window opened in the same directory as the model reader script:
```
pip install pycodestyle
pycodestyle dtm_4D.py
```
The reported errors should be resolved until no errors are given when the second command is executed. For more information on PEP8 standards, see https://peps.python.org/pep-0008/.

Note that the notebooks ending with the string 'timing' were used to produce the timing data used in the Kamodo model reader paper (Figure 10).
.
