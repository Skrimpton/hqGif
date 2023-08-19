# hqGif
A script for ffmpeg to quickly make gifs

[Link to source](https://blog.pkh.me/p/21-high-quality-gif-with-ffmpeg.html)
```zsh
#!/bin/zsh

### Based on this https://blog.pkh.me/p/21-high-quality-gif-with-ffmpeg.html


printHelp()
{
echo -e """—————————————————————————
Usage example:
—————————————————————————

  hqGif ~/Videos/<videofile.ext> -scale 520 -scf bicubic -dir ~/Pictures -name lol

Result:

  ~/Pictures/lol.gif


------------
Explanation
------------

-name

  Sets output filename from original filename
  to given string (e.g. "lol")

  \".gif\" is appended automatically

-dir

  Sets output directory to ~/Pictures

-scale 520

  Makes a 520px scaled gif from ~/Videos/<videofile.ext>

-scf bicubic

  Use bicubic scaling (lancoz by default)


—————————————————————————
Available argument flags:
—————————————————————————

--folder|-dir

    Set output directory


--name|-name|-n|-o

    Set name of output gif

    \".gif\" is appended automatically


--scale|-scale|-s|-sz|-sc|-size

    Set size of output gif
    default is 320px

    ---------------
    IMPORTANT NOTE:
    ---------------

    This script is currently locked to keeping aspect ratio.

    DO NOT USE xxx:xxx


--scale_flag|-scf

    https://trac.ffmpeg.org/wiki/Scaling

    String to set scale flag
    e.g bicub or lancoz

    Has effect on filesize and image quality

    Available flags:

    lanczos (default)
      - The default width (alpha) is 3.
        This can be changed by adding \":param0=\"
        and the desired number.

    area
    bicubic
    bicublin
    bilinear
    experimental
    fast_bilinear
    gauss
    neighbor
    sinc
    spline


--start-time|-st|-ss

    Set start time of videoclip to make gif from


--loops|-loops|-loop

    Set the amout of loops for the gif.

    default is infinite (0)

    -1 no loop (plays once)
    0 infinite loop (default)
    1 loop once (plays 2 times)
    2 loop twice (plays 3 times)
    etc


--duration|-d|-t

    Set duration of clip


--frames-per-second|-fps|--fps

    Set gif fps - higher is smoother, but filesize is larger
    default is 10 fps
"""

}


# --- Check if there are any arguments

if [[ -z $@ ]]
then
    printHelp
    exit 0

else
    originalFile=""
    for var in $@
    do
        if [[ -f $var ]]
        then
            originalFile="$var"
        fi
    done
    if [[ $originalFile = "" || ! -f $originalFile ]]
    then
        echo "No file detected"
        exit 1
    fi
    name="${originalFile##*/}"
    name="${name%.*}"

    start_time=0
    duration=0
    fps=10
    scale=320
    loops=0
    scale_flags="lanczos"
    stats_mode=diff
    # stats_mode=full

    new_folder=""

    # --- Check if there are more arguments than just original file

    while [[ ${#@} > 0 ]]
    do
        if [[ ${1:0:1} = "-" ]]
        then
            case "$1" in
            --start-time|-st|-ss)
                shift
                start_time="$1"
            ;;
            --duration|-d|-t)
                shift
                duration="$1"

            ;;
            --frames-per-second|-fps|--fps)
                shift
                fps="$1"

            ;;
            --scale|-scale|-sc|-size)
                shift
                scale="$1"

            ;;
            --loops|-loops|-loop)
                shift
                loops=$1

            ;;
            --scale-flag|-scf)
                shift
                case "$1" in
                la|lan|lc|lcz|lz)
                    scale_flags="lanczos"
                ;;
                bi|bic)
                    scale_flags="bicubic"
                ;;
                *)
                    scale_flags="$1"
                ;;
                esac

            ;;
            --stats-mode|-sm|-smode)
                shift
                stats_mode="$1"

            ;;
            --filters|-f)
                shift
                filters="$1"
            ;;
            --name|-name|-n|-o)
                shift
                case "$1" in
                *.gif)
                    name="${1%.*}"
                ;;
                *)
                    name="$1"
                ;;
                esac
            ;;
            --help|-h)
                printHelp
                exit 0
            ;;
            --folder|-dir)
                shift
                case "$1" in
                PWD|pwd)
                    new_folder="$PWD"
                ;;
                *)
                    new_folder="$1"
                ;;
                esac
            ;;
            esac
        fi
        shift
    done

    if [[ $new_folder != "" ]]
    then
        if [[ -d "$new_folder" && -w "$new_folder" ]]
        then
            folder="$new_folder"
        else
            echo -e "$new_folder\nis not a valid folder"
            exit 1
        fi
    else
        folder="${originalFile%/*}"
        folder="${folder}"
    fi

    if [[ -d "${folder}" && -w "${folder}" ]]
    then
        trap clean_up 1 2 3 15

        # --- Function to delete the temporary palette

        clean_up()
        {
            # [[ -f $TEMP ]] && echo "TEMP EXISTS" || echo "TEMP DOES NOT EXIST"
            rm "$palette" >/dev/null 2>&1
            # [[ -f $TEMP ]] && echo "TEMP EXISTS" || echo "TEMP DOES NOT EXIST"
        }

        palette=$(mktemp -t $name-$$.XXXXXXXXXX.png)

        if [[ -f $palette ]]
        then
            filters="fps=$fps,scale=$scale:-1:flags=$scale_flags"
            out_file="${folder}/${name}.gif"

            if [[ -f $out_file ]]
            then
                counter=1
                out_new="${out_file%.*} ($counter).gif"
                while [[ -f $out_new ]]
                do
                    ((counter++))
                    out_new="${out_file%.*} ($counter).gif"
                done
                out_file="$out_new"
            fi

            echo -e "\nNew GIF:\n$out_file\n"

            if [[ $start_time = 0 && $duration = 0 ]]
            then
                ffmpeg \
                    -v warning \
                    -i "$originalFile" \
                    -vf "$filters,palettegen=stats_mode=$stats_mode" \
                    -update true  \
                    -y $palette

                ffmpeg \
                    -v warning \
                    -i "$originalFile" \
                    -i $palette \
                    -lavfi "$filters [x]; [x][1:v] paletteuse" \
                    -loop $loops \
                    -y "$out_file"

            elif [[ $start_time > 0 && $duration = 0 ]]
            then
                ffmpeg \
                    -v warning \
                    -ss $start_time \
                    -i "$originalFile" \
                    -vf "$filters,palettegen=stats_mode=$stats_mode" \
                    -update true  \
                    -y $palette

                ffmpeg \
                    -v warning \
                    -ss $start_time \
                    -i "$originalFile" \
                    -i $palette \
                    -lavfi "$filters [x]; [x][1:v] paletteuse" \
                    -loop $loops \
                    -y "$out_file"

            elif [[ $start_time > 0 && $duration > 0 ]]
            then
                ffmpeg \
                    -v warning \
                    -ss $start_time \
                    -t $duration \
                    -i "$originalFile" \
                    -vf "$filters,palettegen=stats_mode=$stats_mode" \
                    -update true  \
                    -y $palette

                ffmpeg \
                    -v warning \
                    -ss $start_time \
                    -t $duration \
                    -i "$originalFile" \
                    -i $palette \
                    -lavfi "$filters [x]; [x][1:v] paletteuse" \
                    -loop $loops \
                    -y "$out_file"
            fi
            clean_up
        else
            echo "Couldn't make palette"
            exit 1
        fi

    else
        echo "Folder is not vald"
        exit 1
    fi
fi
```
