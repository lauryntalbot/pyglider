# PyGlider: Adjust CTD variables

PyGlider applies a post-processing protocol to conductivity, temperature, and salinity variables in NetCDF timeseries files, and subsequently generates NetCDF depth–time grids using Python and `xarray`. The resulting NetCDF files are largely CF-compliant.

The basic workflow consists of converting a NetCDF timeseries into an adjusted timeseries and corresponding depth–time grids. This follows the `binary_to_timeseries` (for Slocum gliders) and `raw_to_timeseries` (for Alseamar gliders) protocols, which convert raw glider data into NetCDF format. The variable `outname` refers to the timeseries output from this prior step.

```python
outname_ctd = pyglider.ncprocess.adjust_CTD(
    outname,
    deploymentyaml,
    l1tsdir,
    griddir,
    dTdC=None,
    tau=None,
    alpha=None,
    maskfunction=None,
    interp_variables=None
) 
```

Data are read from and written to directories, and metadata are supplied via a YAML file.

## Post-processing steps within `pyglider.ncprocess.adjust_CTD`

### 1. Identify anomalous conductivity values 

We identify and flag conductivity values that are clearly unphysical, typically caused by air bubbles in the conductivity cell.

A two-step statistical criterion is applied:
\begin{itemize}
\item First, data points more than \textbf{5 standard deviations} from the mean are temporarily flagged within each depth and profile bin.
\item The mean and standard deviation are then recomputed excluding these points.
\item Values still exceeding \textbf{3 standard deviations} from the recomputed mean are flagged as \textbf{bad (QC = 4)} in \texttt{conductivity_QC}.
\end{itemize}

However, if the deviation is smaller than the sensor accuracy (0.0003~S/m for the GPCTD), the data are retained.

This procedure is applied using:
\begin{itemize}
\item Profile bins of 50 profiles
\item Depth bins of 5~m
\end{itemize}

Using profile-index binning (rather than time or temperature) helps isolate unphysical values.

Salinity (\texttt{salinity_QC}) is flagged as bad (QC = 4) wherever \texttt{conductivity_QC} is QC4.


### 2. Determine what dTdC, tau, and alpha are used in the correction 
We correct for:
\begin{itemize}
\item Sensor misalignment between temperature and conductivity (\texttt{dTdC})
\item Thermal lag effects (\texttt{tau}, \texttt{alpha})
\end{itemize}

Where:
\begin{itemize}
\item \texttt{dTdC} = time lag (seconds) between temperature and conductivity sensors
\item \texttt{tau} = thermal response time constant (seconds)
\item \texttt{alpha} = scaling of thermal coupling between water and the conductivity cell
\end{itemize}

Further details and methods for determining these parameters are available at:\
\url{https://cproof.uvic.ca/gliderdata/deployments/reports/}

For recent C-PROOF missions, these values are included in the YAML file. However, users may:
\begin{itemize}
\item Provide custom values for \texttt{dTdC}, \texttt{tau}, and \texttt{alpha}, or
\item Skip corrections by setting parameters to \texttt{None}
\end{itemize}

If user-supplied values differ from those in the YAML file, a warning is issued, but the user-provided values are used.

New variables introduced:
\begin{itemize}
\item \texttt{temperature_adjusted}
\item \texttt{salinity_adjusted}
\item \texttt{temperature_adjusted_QC}
\item \texttt{salinity_adjusted_QC}
\end{itemize}
 
 ### 3. Recalculate derived variables
Using TEOS-10, we recompute:
\begin{itemize}
\item \texttt{potential_density_adjusted}
\item \texttt{potential_temperature_adjusted}
\end{itemize}

Their corresponding QC variables:
\begin{itemize}
\item \texttt{potential_density_adjusted_QC}
\item \texttt{potential_temperature_adjusted_QC}
\end{itemize}

These are flagged as bad (QC = 4) wherever \texttt{salinity_adjusted_QC} is QC4.

### 4. Convert the adjusted NetCDF timeseries to a NetCDF depth-time grid. 

The adjusted timeseries is converted into a gridded NetCDF dataset.

\begin{itemize}
\item Default binning:
\begin{itemize}
\item 1~m depth bins
\item Profile bins
\end{itemize}
\item Variables are averaged within each bin.
\end{itemize}

QC variables use a \texttt{QC_protocol} that selects the maximum QC value within each bin, ensuring that bad data are not diluted.

Optional parameters:
\begin{itemize}
\item \texttt{maskfunction}
\item \texttt{interp_variables}
\end{itemize}

C-PROOF applies:
\begin{itemize}
\item \texttt{pyglider.ncprocess.CPROOF_mask} to exclude QC4 data from gridded products
\item \texttt{pyglider.ncprocess.interpolate_vertical} to interpolate over vertical gaps up to 50~m
\end{itemize}