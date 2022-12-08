---
title: "WQP Inventory Problem"
output: html_document
date: "2022-11-29"
---

``` r
library(tidyverse)
library(dataRetrieval)


# Load necessary data structures ------------------------------------------

wqp_state_codes <- structure(list(value = c("US:00", "US:01", "US:02", "US:04", 
                                        "US:05", "US:06", "US:08", "US:09", "US:10", "US:11", "US:12", 
                                        "US:13", "US:15", "US:16", "US:17", "US:18", "US:19", "US:20", 
                                        "US:21", "US:22", "US:23", "US:24", "US:25", "US:26", "US:27", 
                                        "US:28", "US:29", "US:30", "US:31", "US:32", "US:33", "US:34", 
                                        "US:35", "US:36", "US:37", "US:38", "US:39", "US:40", "US:41", 
                                        "US:42", "US:44", "US:45", "US:46", "US:47", "US:48", "US:49", 
                                        "US:50", "US:51", "US:53", "US:54", "US:55", "US:56"),
                              name = c("Unspecified", 
                                       "Alabama", "Alaska", "Arizona", "Arkansas", "California", "Colorado", 
                                       "Connecticut", "Delaware", "District of Columbia", "Florida", 
                                       "Georgia", "Hawaii", "Idaho", "Illinois", "Indiana", "Iowa", 
                                       "Kansas", "Kentucky", "Louisiana", "Maine", "Maryland", "Massachusetts", 
                                       "Michigan", "Minnesota", "Mississippi", "Missouri", "Montana", 
                                       "Nebraska", "Nevada", "New Hampshire", "New Jersey", "New Mexico", 
                                       "New York", "North Carolina", "North Dakota", "Ohio", "Oklahoma", 
                                       "Oregon", "Pennsylvania", "Rhode Island", "South Carolina", "South Dakota", 
                                       "Tennessee", "Texas", "Utah", "Vermont", "Virginia", "Washington", 
                                       "West Virginia", "Wisconsin", "Wyoming")),
                         row.names = c(NA, 
                                       -52L),
                         class = "data.frame")

wqp_states <- c("Alabama", "Alaska", "Arizona", "Arkansas", "California", "Colorado", 
                "Connecticut", "Delaware", "District of Columbia", "Florida", 
                "Georgia", "Hawaii", "Idaho", "Illinois", "Indiana", "Iowa", 
                "Kansas", "Kentucky", "Louisiana", "Maine", "Maryland", "Massachusetts", 
                "Michigan", "Minnesota", "Mississippi", "Missouri", "Montana", 
                "Nebraska", "Nevada", "New Hampshire", "New Jersey", "New Mexico", 
                "New York", "North Carolina", "North Dakota", "Ohio", "Oklahoma", 
                "Oregon", "Pennsylvania", "Rhode Island", "South Carolina", "South Dakota", 
                "Tennessee", "Texas", "Utah", "Vermont", "Virginia", "Washington", 
                "West Virginia", "Wisconsin", "Wyoming")

wqp_codes <- list(
  characteristicName = list(
    tss = c("Total suspended solids", 
            "Suspended sediment concentration (SSC)", "Suspended Sediment Concentration (SSC)", 
            "Total Suspended Particulate Matter", "Fixed suspended solids"),
    chlorophyll = c("Chlorophyll", "Chlorophyll A", "Chlorophyll a", 
                    "Chlorophyll a (probe relative fluorescence)", "Chlorophyll a (probe)", 
                    "Chlorophyll a - Periphyton (attached)", "Chlorophyll a - Phytoplankton (suspended)", 
                    "Chlorophyll a, corrected for pheophytin", "Chlorophyll a, free of pheophytin", 
                    "Chlorophyll a, uncorrected for pheophytin", "Chlorophyll b", 
                    "Chlorophyll c", "Chlorophyll/Pheophytin ratio"),
    secchi = c("Depth, Secchi disk depth", 
               "Depth, Secchi disk depth (choice list)", "Secchi Reading Condition (choice list)", 
               "Secchi depth", "Water transparency, Secchi disc"),
    cdom = "Colored dissolved organic matter (CDOM)", 
    doc = c("Organic carbon", "Total carbon", "Hydrophilic fraction of organic carbon", 
            "Non-purgeable Organic Carbon (NPOC)")),
  sampleMedia = c("Water", 
                  "water"),
  siteType = c("Lake, Reservoir, Impoundment", "Stream", 
               "Estuary", "Facility")
)


# Do some data prep -------------------------------------------------------

# Convert states list to FIPS list
state_codes <- wqp_state_codes %>%
  filter(name %in% wqp_states) %>%
  pull(value)

# Identify available constituent sets
constituents <- names(wqp_codes$characteristicName)

# Prepare the args to whatWQPdata. All arguments will be the same every time
# except characteristicName, which we'll loop through to get separate counts
# for each
wqp_args <- list(
  statecode = state_codes,
  siteType = wqp_codes$siteType,
  # To be updated each time through loop:
  characteristicName = NA,
  sampleMedia = wqp_codes$sampleMedia
  # We'd include dates, but they get ignored by the service behind whatWQPdata
)


# WQP data info calls -----------------------------------------------------

# The goal is to iterate through the WQP codes in the loop that's below `test`
# Here's an example of something that works (doesn't include WQP codes though):
test <- whatWQPdata(statecode = "US:08",
            siteType = c("Lake, Reservoir, Impoundment", "Stream", "Estuary", "Facility"),
            sampleMedia = c("Water", "water"))

head(test)
#>                        total_type      lat       lon ProviderName
#> 1 FeatureCollection Feature Point 40.49331 -106.4498         NWIS
#> 2 FeatureCollection Feature Point 40.52304 -106.3692         NWIS
#> 3 FeatureCollection Feature Point 40.55748 -106.3900         NWIS
#> 4 FeatureCollection Feature Point 40.63553 -106.3967         NWIS
#> 5 FeatureCollection Feature Point 40.55137 -106.6164         NWIS
#> 6 FeatureCollection Feature Point 40.57331 -106.5100         NWIS
#>   OrganizationIdentifier             OrganizationFormalName
#> 1                USGS-CO USGS Colorado Water Science Center
#> 2                USGS-CO USGS Colorado Water Science Center
#> 3                USGS-CO USGS Colorado Water Science Center
#> 4                USGS-CO USGS Colorado Water Science Center
#> 5                USGS-CO USGS Colorado Water Science Center
#> 6                USGS-CO USGS Colorado Water Science Center
#>   MonitoringLocationIdentifier                   MonitoringLocationName
#> 1                USGS-06611100           GRIZZLY CREEK NEAR SPICER, CO.
#> 2                USGS-06611200           BUFFALO CREEK NEAR HEBRON, CO.
#> 3                USGS-06611300           GRIZZLY CREEK NEAR HEBRON, CO.
#> 4                USGS-06611500            GRIZZLY CREEK NEAR WALDEN, CO
#> 5                USGS-06611700  LITTLE GRIZZLY CREEK NEAR COALMONT, CO.
#> 6                USGS-06611800 LITTLE GRIZZLY CREEK ABOVE COALMONT, CO.
#>   MonitoringLocationTypeName ResolvedMonitoringLocationTypeName
#> 1                     Stream                             Stream
#> 2                     Stream                             Stream
#> 3                     Stream                             Stream
#> 4                     Stream                             Stream
#> 5                     Stream                             Stream
#> 6                     Stream                             Stream
#>   HUCEightDigitCode
#> 1          10180001
#> 2          10180001
#> 3          10180001
#> 4          10180001
#> 5          10180001
#> 6          10180001
#>                                                                siteUrl
#> 1 https://www.waterqualitydata.us/provider/NWIS/USGS-CO/USGS-06611100/
#> 2 https://www.waterqualitydata.us/provider/NWIS/USGS-CO/USGS-06611200/
#> 3 https://www.waterqualitydata.us/provider/NWIS/USGS-CO/USGS-06611300/
#> 4 https://www.waterqualitydata.us/provider/NWIS/USGS-CO/USGS-06611500/
#> 5 https://www.waterqualitydata.us/provider/NWIS/USGS-CO/USGS-06611700/
#> 6 https://www.waterqualitydata.us/provider/NWIS/USGS-CO/USGS-06611800/
#>   activityCount resultCount StateName     CountyName
#> 1            38        1384  Colorado Jackson County
#> 2            38        1380  Colorado Jackson County
#> 3            36        1368  Colorado Jackson County
#> 4             1          20  Colorado Jackson County
#> 5            62          62  Colorado Jackson County
#> 6            37        1375  Colorado Jackson County

# Loop over the constituents, getting rows for each. With tryCatch I thought this
# would continue but the 504 error seems to be wrecking the attempts?
sample_time <- system.time({
  samples <- bind_rows(lapply(constituents, function(constituent) {
    
    message(Sys.time(), ": getting inventory for ", constituent)
    
    wqp_args$characteristicName <- wqp_codes$characteristicName[[constituent]]
    
    tryCatch({
      wqp_wdat <- do.call(whatWQPdata, wqp_args)
      mutate(wqp_wdat, constituent = constituent)
    }, error = function(e) {
      # Keep going IFF the only error was that there weren't any matching sites
      if(grepl("arguments imply differing number of rows", e$message)) {
        NULL
      } else {
        stop(e)
      }
    })
  }))
})
#> 2022-11-29 11:33:09: getting inventory for tss
#> Request failed [504]. Retrying in 1.8 seconds...
#> Request failed [504]. Retrying in 1.1 seconds...
#> For: https://www.waterqualitydata.us/data/Station/search?statecode=US%3A01%3BUS%3A02%3BUS%3A04%3BUS%3A05%3BUS%3A06%3BUS%3A08%3BUS%3A09%3BUS%3A10%3BUS%3A11%3BUS%3A12%3BUS%3A13%3BUS%3A15%3BUS%3A16%3BUS%3A17%3BUS%3A18%3BUS%3A19%3BUS%3A20%3BUS%3A21%3BUS%3A22%3BUS%3A23%3BUS%3A24%3BUS%3A25%3BUS%3A26%3BUS%3A27%3BUS%3A28%3BUS%3A29%3BUS%3A30%3BUS%3A31%3BUS%3A32%3BUS%3A33%3BUS%3A34%3BUS%3A35%3BUS%3A36%3BUS%3A37%3BUS%3A38%3BUS%3A39%3BUS%3A40%3BUS%3A41%3BUS%3A42%3BUS%3A44%3BUS%3A45%3BUS%3A46%3BUS%3A47%3BUS%3A48%3BUS%3A49%3BUS%3A50%3BUS%3A51%3BUS%3A53%3BUS%3A54%3BUS%3A55%3BUS%3A56&siteType=Lake%2C%20Reservoir%2C%20Impoundment%3BStream%3BEstuary%3BFacility&characteristicName=Total%20suspended%20solids%3BSuspended%20sediment%20concentration%20%28SSC%29%3BSuspended%20Sediment%20Concentration%20%28SSC%29%3BTotal%20Suspended%20Particulate%20Matter%3BFixed%20suspended%20solids&sampleMedia=Water%3Bwater&zip=yes&mimeType=geojson
#> Gateway Timeout (HTTP 504).
#> Error in UseMethod("mutate"): no applicable method for 'mutate' applied to an object of class "NULL"
#> Timing stopped at: 0.05 0.015 183.5
```

<sup>Created on 2022-11-29 with [reprex v2.0.2](https://reprex.tidyverse.org)</sup>