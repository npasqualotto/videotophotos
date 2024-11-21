<p xmlns:cc="http://creativecommons.org/ns#" >This work is licensed under <a href="https://creativecommons.org/licenses/by-nc/4.0/?ref=chooser-v1" target="_blank" rel="license noopener noreferrer" style="display:inline-block;">CC BY-NC 4.0<img style="height:22px!important;margin-left:3px;vertical-align:text-bottom;" src="https://mirrors.creativecommons.org/presskit/icons/cc.svg?ref=chooser-v1" alt=""><img style="height:22px!important;margin-left:3px;vertical-align:text-bottom;" src="https://mirrors.creativecommons.org/presskit/icons/by.svg?ref=chooser-v1" alt=""><img style="height:22px!important;margin-left:3px;vertical-align:text-bottom;" src="https://mirrors.creativecommons.org/presskit/icons/nc.svg?ref=chooser-v1" alt=""></a></p>

# R code to break camera trap videos into images (.jpg files)

[Explain in more details what my code do...]

``` {r} 
## loading packages
library(av)
library(exiftoolr)
library(filesstrings)

## setting your main directory (folder of your camera trap project)
# here, let's work with 'project_yyy' stored in this repository
main_dir <- "./project_yyy"

# getting paths with videos
# camera traps (bushnell model 119949C) create video files with '.MOV' extension
all_paths <- list.dirs(main_dir, full.names = T, recursive= T)
video_paths <- list.files(path = all_paths, pattern = "\\.MOV$", full.names = T)

# getting all video file names
videos_names <- basename(video_paths)

# 1st loop corresponds to one iteration for each video
for (i in 1:length(video_paths)) {
  # extracting imgs from video (1 frame per second)
  # storing new images in a temporary folder
  av_video_images(video_paths[i], destdir = paste0(dirname(video_paths[i]), "/temp"), format = "jpg", fps = 1)
  # listing new image paths
  newimg_paths <- list.files(paste0(dirname(video_paths[i]), "/temp"), full.names = T)
  # rename new images based on the video they came from
  file.rename(newimg_paths, sub('/temp/', paste0('/temp/', sub('\\.MOV$', '', videos_names[i]), "_"), newimg_paths))
  # extracting original date and time from video
  exifinfo_video <- exif_read(video_paths[i], tags = c("FileName", "CreateDate", "MediaDuration"))
  # creating corrected date and time based on video metadata info
  # first, let's create a character vector by repeating date and time of the video with same lenght as number of imgs extracted from it
  # the repeated date and time obtained from 'CreateDate' metadata field correspond to the video first instant
  # transform the character vector in a POSIXct object
  # sum 1s after the first date time element to get the correct date and time of each frame  
  # NOTICE: the second that a video starts (metadata info, field CreateDate) may differ from the second imprinted in the video (bottom of the video)
  # so, a small variation (1s error) is expected
  correct_dates <- rep(exifinfo_video$CreateDate, length(newimg_paths))
  correct_dates <- (as.POSIXct(correct_dates, tz = "America/Sao_Paulo", format='%Y:%m:%d %H:%M:%S'))
  correct_dates <- correct_dates + seq(0, length(newimg_paths)-1)
  # updating (renamed) image paths
  newimg_paths <- list.files(paste0(dirname(video_paths[i]), "/temp"), full.names = T)
  # nestled loop to correct date and time of each image from 1st video 
  for (j in 1:length(newimg_paths)) {
    exif_call(args = paste0("-CreateDate=\"", correct_dates[j], "\""), path= newimg_paths[j])
  }
  # print image paths and their corrected dates in R console
  print(exif_read(newimg_paths, tags = c("filename", "CreateDate")))
  # moving corrected images to the directory where videos are stored
  move_files(newimg_paths, dirname(video_paths[i]))
  # deleting temp folder after each i cycle
  dirs <- list.dirs(main_dir)
  temp_dir <- dirs[grep("temp", dirs)]
  unlink(temp_dir, recursive = T)
}

```