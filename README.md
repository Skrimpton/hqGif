# hqGif
A script for ffmpeg to quickly make gifs

[Link to source](https://blog.pkh.me/p/21-high-quality-gif-with-ffmpeg.html)
```zsh
#!/bin/sh

#-------------------------------------------------------------------------------
# Please feel free to edit, reproduce, and improve this script — in any way.
#-------------------------------------------------------------------------------

# Dependencies                       : ffmpeg

# Based on + borrows from this    : https://blog.pkh.me/p/21-high-quality-gif-with-ffmpeg.html
# and this                           : https://superuser.com/a/1323430

# Works in bash and zsh, probably more.

# -------------------------------------------------------
#   EDIT THE FOLLOWING TO CHANGE DEFAULT SETTINGS
# -------------------------------------------------------

#   START_TIME ----------------------------

    start_time=0

# - DURATION ------------------------------

    duration=0

# - FPS -----------------------------------

    fps=10

# - SCALE ---------------------------------

    scale=320

# - LOOPS ---------------------------------

    loops=0

# - SCALE_FLAGS ---------------------------

    scale_flags="lanczos"

# - STATS_MODE ----------------------------

    stats_mode=diff
#     stats_mode=full

# - DITHER --------------------------------

    dither="none"   # Avoid using dither in final command, even if set to none. if not explicitly asked for.
                    # Noticably degraded quality when I tested on higher resolution video.

#     dither="sierra2_4a"
#     dither="bayer:bayer_scale=5"

# - DIFFMODE ------------------------------

    diff_mode="none"
#     diff_mode="rectangle"

# - IN_RAM --------------------------------

#     in_ram=0      #   <in_ram=0>  genrates a png using mktemp.
    in_ram=1        #   <in_ram=1>  does it in RAM
                    #   Be careful with duration of video — CAN USE A LOT OF MEMORY!

#-------------------------------------------------------------------


#-------------------------------------------------------------------
# TODO :
#   - Add filetype/extension-filter case statement for inputfiles?
#-------------------------------------------------------------------

IFS="" # --- Disabling IFS for filenames


printExample()
{
echo """—————————————————————————
Simple usage example
—————————————————————————
    hqGif ~/Videos/<videofile.ext>

Result:

    hqGif ~/Videos/<videofile.gif>

------------
Explanation
------------

    Just a file will create a GIF in the same folder
    using the current default settings of:

    start_time=$start_time
    duration=$duration
    fps=$fps
    scale=$scale
    loops=$loops
    scale_flags=$scale_flags
    stats_mode=$stats_mode
    in_ram=$in_ram
    dither=$dither
    diff_mode=$diff_mode

---
BTW
---
    --ram|-ram

      Switches to opposite of current default ram state


    --ram-on|-ram-on

      Sets in_ram to 1
      !!! CAN USE A LOT OF MEMORY !!!


    --ram-off|-ram-off|-palette|--palette

        Genrates a png using mktemp.


—————————————————————————
Complex Usage example:
—————————————————————————

    hqGif ~/Videos/<videofile.ext> -scale 520 -scf bicubic -dir ~/Pictures -name lol

Result:

    ~/Pictures/lol.gif


------------
Explanation
------------

    -name

        Sets output filename from original filename
        to given string (e.g. \"lol\")

        \".gif\" is appended automatically

    -dir

        Sets output directory to ~/Pictures

    -scale 520

        Makes a 520px scaled gif from ~/Videos/<videofile.ext>

    -scf bicubic

        Use bicubic scaling (lancoz by default)

"""
}

printHelp() # --- function() : Help-text
{
echo """For links it's recommended you use --ram-on.
This avoids double-downloading.

--duration|-d|-t

    Set duration of clip


--diff-mode|-diff-mode|-dmode|-dm

    Sets diff_mode to 'rectangle'

    https://www.ffmpeg.org/ffmpeg-filters.html#paletteuse
    'rectangle'

    If set, define the zone to process

        Only the changing rectangle will be reprocessed.
        This is similar to GIF cropping/offsetting compression mechanism.

        This option can be useful for speed if only a part of the image is changing,
        and has use cases such as limiting the scope of the error diffusal dither to the rectangle that bounds the moving scene

        (it leads to more deterministic output if the scene doesn’t change much,
        and as a result less moving noise and better GIF compression).

    Default is $diff_mode.


--dither|-dither|-dit|-di|-dt

    https://www.ffmpeg.org/ffmpeg-filters.html#paletteuse

    Select dithering mode. Available algorithms are:
    (prefix with -- or - to use as argument)

    bayer
        Ordered 8x8 bayer dithering (deterministic)

    heckbert
        Dithering as defined by Paul Heckbert in 1982 (simple error diffusion).
        Note: this dithering is sometimes considered \"wrong\" and is included as a reference.

    floyd_steinberg
        Floyd and Steingberg dithering (error diffusion)

    sierra2
        Frankie Sierra dithering v2 (error diffusion)

    sierra2_4a
        Frankie Sierra dithering v2 \"Lite\" (error diffusion)

    sierra3
        Frankie Sierra dithering v3 (error diffusion)

    burkes
        Burkes dithering (error diffusion)

    atkinson
        Atkinson dithering by Bill Atkinson at Apple Computer (error diffusion)

    none
        Disable dithering.

    Default is $dither


--example|-example

    Show examples and explanations


--folder|-dir

    Set output directory


--frames-per-second|-fps|--fps

    Set gif fps - higher is smoother, but filesize is larger
    default is 10 fps


--loops|-loops|-loop

    Set the amout of loops for the gif.

    default is infinite (0)

    -1 no loop (plays once)
    0 infinite loop (default)
    1 loop once (plays 2 times)
    2 loop twice (plays 3 times)
    etc


--name|-name|-n|-o

    Set name of output gif

    \".gif\" is appended automatically


--ram|-ram

    Switches to opposite of current default ram state


--ram-off|-ram-off|-palette|--palette

    Creates palette .png using mktemp


--ram-on|-ram-on

    Creates palette in RAM

    NB: This can use a lot of memory

    ffmpeg-command is the one from:
    https://superuser.com/a/1323430


--scale|-scale|-size|-sz|-sc|-s

    Set size of output gif
    default is 320px

    ---------------
    IMPORTANT NOTE:
    ---------------

    This script is currently locked to keeping aspect ratio.

    DO NOT USE xxx:xxx


--scale-flags|--sflags|-sflags|-scf|-sf

    https://trac.ffmpeg.org/wiki/Scaling

    String to set scale flag
    e.g bicub or lancoz

    Has effect on filesize and image quality

    Available flags:
    (prefix with -- or - to use as argument)

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

    Set start time to use from video

    e.g. in seconds or < hh:mm:ssSEPff >
    https://ffmpeg.org/ffmpeg-utils.html#time-duration-syntax
"""

}


# --- Check if there are any arguments

if [[ -z $@ ]]
then
    printHelp
    exit 0

else

    palette_use=""
    dither_flag=0


    scale_flag_counter=0
    dither_flag_counter=0

    use_custom_filters=0
    originalFile=""
    new_folder=""
    name=""
    has_url=0
    passed_name=0

#     oldifs="$IFS"
    for var in $@
    do
        regex='^(https?|ftp|file)://[-A-Za-z0-9\+&@#/%?=~_|!:,.;]*[-A-Za-z0-9\+&@#/%=~_|]\.[-A-Za-z0-9\+&@#/%?=~_|!:,.;]*[-A-Za-z0-9\+&@#/%=~_|]$' # https://stackoverflow.com/a/55267709


        if [[ -f $var ]]
        then
            originalFile="$var"

        elif [[ $var =~ $regex ]]
        then
            originalFile="$var"
            has_url=1
        fi
    done
    if [[ "$originalFile" = "" || ! -f "$originalFile" ]] && [[ $has_url = 0 ]]
    then

        echo "No file detected"
        exit 1
    fi

    name="${originalFile##*/}"
    name="${name%.*}"

    # --- Check if there are more arguments than just original file


    nameIsWrong() # --- function() : check if given name is bad
    {
        case "$(echo "$1" | xargs)" in
            /*)
                true
            ;;

            # ------------------------------
            # Invalid name on dash at start.
            # ------------------------------
            # ... I know, but you don't wanna name it after commands either, if you mess up and forget to type a name.
            # so, at your own peril, just comment out the following bit if you want dashes in the start of output filename

            -*)
                true
            ;;

            # This bit ^ -------------------

            *)
                [[ -z $1 ]] && true || false

            ;;
        esac
    }

    isWrong() # --- function() : check if given variable starts with -
    {
        case "$(echo "$1" | xargs)" in
            -*)
                true
            ;;
            *)
                [[ -z $1 ]] && true || false

            ;;
        esac
    }

    die() # --- function() : exit the script
    {
        echo "Check arguments:"
        echo "Something is wrong with $1. exiting..."
        exit 1
    }

    while [[ ${#@} > 0 ]]
    do
        if [[ ${1:0:1} = "-" ]]
        then

            case "$1" in
            --start-time|-st|-ss)
                arg="$1"
                if isWrong "$2";then die "$arg";fi
                shift
                start_time="$1"
            ;;
            --diff-mode|-diff-mode|-dmode|-dm)
                diff_mode="rectangle"
            ;;
            --bayer|-bayer)
                dither="bayer"
                ((dither_flag_counter++))
                dither_flag=1
            ;;
            --heckbert|-heckbert)
                dither="heckbert"
                ((dither_flag_counter++))
                dither_flag=1
            ;;
            --floyd_steinberg|-floyd_steinberg)
                dither="floyd_steinberg"
                ((dither_flag_counter++))
                dither_flag=1
            ;;
            --sierra2|-sierra2)
                dither="sierra2"
                ((dither_flag_counter++))
                dither_flag=1
            ;;
            --sierra2_4a|-sierra2_4a)
                dither="sierra2_4a"
                ((dither_flag_counter++))
                dither_flag=1
            ;;
            --sierra3|-sierra3)
                dither="sierra3"
                ((dither_flag_counter++))
                dither_flag=1
            ;;
            --burkes|-burkes)
                dither="burkes"
                ((dither_flag_counter++))
                dither_flag=1
            ;;
            --atkinson|-atkinson)
                dither="atkinson"
                ((dither_flag_counter++))
                dither_flag=1
            ;;
            --dither-off|-doff)
                dither="none"
                ((dither_flag_counter++))
                dither_flag=1
            ;;
            --dither|-dither|-dit|-di|-dt)
                arg="$1"
                if isWrong "$2";then die "$arg";exit 1;fi
                shift
                if [[ $1 != "none" ]]
                then
                    dither="$1"
                    dither_flag=1
                fi

            ;;
            --duration|-d|-t)
                arg="$1"
                if isWrong "$2";then echo "$arg : Can't have a negative duration";exit 1;fi
                shift
                duration="$1"

            ;;
            --frames-per-second|-fps|--fps)
                arg="$1"
                if isWrong "$2";then echo "$arg : GIFs can't have negative frames";exit 1;fi
                shift
                fps="$1"

            ;;
            --scale|-scale|-sc|-size|-sz|-s)
                arg="$1"
                if isWrong "$2";then die "$arg";fi
                shift
                [[ $1 < 128 ]] && echo "What is this, a gif for ants!?"
                scale="$1"

            ;;
            --loops|-loops|-loop)
                arg="$1"
                if isWrong "$2";then die "$arg";fi
                shift
                loops=$1

            ;;
            --lanczos|-lanczos)
                scale_flags="lanczos"
                ((scale_flag_counter++))
            ;;
            --area|-area)
                scale_flags="area"
                ((scale_flag_counter++))
            ;;
            --bicubic|-bicubic)
                scale_flags="bicubic"
                ((scale_flag_counter++))
            ;;
            --bicublin|-bicublin)
                scale_flags="bicublin"
                ((scale_flag_counter++))
            ;;
            --bilinear|-bilinear)
                scale_flags="bilinear"
                ((scale_flag_counter++))
            ;;
            --experimental|-experimental)
                scale_flags="experimental"
                ((scale_flag_counter++))
            ;;
            --fast-bilinear|-fast-bilinear)
                scale_flags="fast_bilinear"
                ((scale_flag_counter++))
            ;;
            --gauss|-gauss)
                scale_flags="gauss"
                ((scale_flag_counter++))
            ;;
            --neighbor|-neighbor)
                scale_flags="neighbor"
                ((scale_flag_counter++))
            ;;
            --sinc|-sinc)
                scale_flags="sinc"
                ((scale_flag_counter++))
            ;;
            --spline|-spline)
                scale_flags="spline"
                ((scale_flag_counter++))
            ;;
            --scale-flags|-scf|-sflags|--sflags|-sf)
                arg="$1"
                if isWrong "$2";then die "$arg";fi
                shift
                ((scale_flag_counter++))
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
                arg="$1"
                if isWrong "$2";then die "$arg";fi
                shift
                stats_mode="$1"

            ;;
            --filters|-f)
                arg="$1"
                if isWrong "$2";then die "$arg";fi
                shift
                use_custom_filters=1
                filters="$1"
            ;;
            --name|-name|-n|-o)
                arg="$1"
                if nameIsWrong "$2";then die "$arg";fi
                shift
                case "$1" in
                *.gif)
                    name="${1%.*}"
                ;;
                *)
                    name="$1"
                ;;
                esac
                passed_name=1
            ;;
            --ram|-ram)
                if [[ $in_ram = 0 ]]
                then
                    echo "Using RAM palette"
                    in_ram=1
                else
                    echo "Using PNG palette"
                    in_ram=0
                fi
            ;;
            --ram-on|-ram-on)
                echo "Using RAM palette"
                in_ram=1
            ;;
            --ram-off|-ram-off|-palette|--palette)
                echo "Using PNG palette"
                in_ram=0
            ;;
            --help|-h)
                printHelp
                exit 0
            ;;
            --example|-example)
                printExample
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
            *)
                if [[ ! -f $1 ]] # Check for and exit if illegal command.
                then
                    echo -e "A crime has been comitted!\nIllegal command: $1\nUse -h, --help, or no arguments to print help"
                    exit 1
                fi
            ;;
            esac
        fi
        shift
    done

    if [[ $has_url = 1 ]]
    then
#         if [[ $passed_name = 0 ]]
#         then
# #             echo "Have to use --name when using URLs"
# #             exit 1
#         fi
        if [[ $new_folder = "" ]]
        then
            new_folder="$PWD"
        fi
    fi

    if [[ $dither_flag = 1 ]]
    then
        palette_use="paletteuse=dither=$dither:diff_mode=$diff_mode"
    else
        palette_use="paletteuse=diff_mode=$diff_mode"
    fi

    if [[ $scale_flag_counter > 1 ]]
    then
        echo -e "\nToo many scale-flags!\nOnly using $scale_flags"
    fi
    if [[ $dither_flag_counter > 1 ]]
    then
        echo -e "\nToo many dither algorithms passed!\nOnly using $dither"
    fi

    if [[ $new_folder != "" ]] # Check for and exit if illegal new_folder.
    then
        if [[ -d "$new_folder" && -w "$new_folder" ]]
        then
            folder="$new_folder"
        else
            echo -e "$new_folder\n... is not a valid folder"
            exit 1
        fi

    else # Assign folder based on input file if no given folder
        folder="${originalFile%/*}"
        folder="${folder}"
    fi

    if [[ -d "${folder}" && -w "${folder}" ]] # --- Verify folder.
    then

        setFiltersAndOutfile() # --- function() to set $filters and $out_file.
        {
            if [[ $use_custom_filters = 0 ]]
            then
                filters="fps=$fps,scale=$scale:-1:flags=$scale_flags"
            else
                [[ $filters = "" ]] && exit 1 && echo "No custom filter set"
            fi
            out_file="${folder}/${name}.gif"

            if [[ -f $out_file ]] # If filename conflict of $out_file: append " (x)"
            then
                counter=1
                out_new="${out_file%.*} ($counter).gif"

                until [[ ! -f $out_new ]] # Verify available new name, and make another until no conflict
                do
                    ((counter++))
                    out_new="${out_file%.*} ($counter).gif"
                done

                out_file="$out_new"
            fi

        }

        notifyStatus()
        {
            if [[ $1 = 0 ]]
            then
                echo -e "\n\e[1;44;1;33mNEW GIF:\e[m\n\e[1;33m$out_file\e[m\n\n$filters"
                echo "stats_mode=$stats_mode"
                echo "$palette_use"
            else
                echo -e "\n\e[1;41;1;34mERROR:\e[m\n\e[1;33m$out_file\e[m\n\n$filters"
                echo "stats_mode=$stats_mode"
                echo "$palette_use"
            fi
        }

        if [[ $in_ram = 0 ]]
        then
            # ------------------------------------------------
            #   Make temporary palette and trap clean_up.
            # ------------------------------------------------
            #
            #   Not sure if better/healthier to skip removal all together,
            #   given the very low potential for filename conflicts
            #   ... *if* the script gets run concurrently.
            #
            #   On the other hand the palette .png(s) could in some cases be quite large?
            #
            # ------------------------------------------------

            trap clean_up 1 2 3 15
            palette="$(mktemp -t "$name"-$$.XXXXXXXXXX.png)"

            clean_up() # --- function() : delete the temporary palette (clean_up)
            {
                rm "$palette" >/dev/null 2>&1
            }

            # ------------------------------------------------

            if [[ -f $palette ]] # --- Verify palette's existence
            then

                setFiltersAndOutfile
                # ---  Ugly if-elif-heck to execute based on start_time and duration

                    if [[ $start_time = 0 && $duration = 0 ]]
                    then
                        ffmpeg \
                            -v warning \
                            -i "$originalFile" \
                            -vf "$filters, palettegen=stats_mode=$stats_mode" \
                            -update true  \
                            -y "$palette"

                        ffmpeg \
                            -v warning \
                            -i "$originalFile" \
                            -i "$palette" \
                            -filter_complex \
                                "$filters [x]; \
                                [x][1:v] $palette_use" \
                            -loop $loops \
                            -n "$out_file"

                            notifyStatus $?

                    elif [[ $start_time > 0 && $duration = 0 ]]
                    then
                        ffmpeg \
                            -v warning \
                            -ss $start_time \
                            -i "$originalFile" \
                            -vf "$filters,palettegen=stats_mode=$stats_mode" \
                            -update true  \
                            -y "$palette"

                        ffmpeg \
                            -v warning \
                            -ss $start_time \
                            -i "$originalFile" \
                            -i "$palette" \
                            -filter_complex \
                                "$filters [x]; \
                                [x][1:v] $palette_use" \
                            -loop $loops \
                            -n "$out_file"

                            notifyStatus $?

                    elif [[ $start_time > 0 && $duration > 0 ]]
                    then
                        ffmpeg \
                            -v warning \
                            -ss $start_time \
                            -t $duration \
                            -i "$originalFile" \
                            -vf "$filters,palettegen=stats_mode=$stats_mode" \
                            -update true  \
                            -y "$palette"

                        ffmpeg \
                            -v warning \
                            -ss $start_time \
                            -t $duration \
                            -i "$originalFile" \
                            -i "$palette" \
                            -filter_complex \
                                "$filters [x]; \
                                [x][1:v] $palette_use" \
                            -loop $loops \
                            -n "$out_file"

                            notifyStatus $?
                    fi

                    clean_up

            else
                echo "Couldn't make palette"
                exit 1
            fi

        else # Do the whole ugly thing in memory : https://superuser.com/a/1323430
            setFiltersAndOutfile
            if [[ $start_time = 0 && $duration = 0 ]]
            then
                ffmpeg \
                -i "$originalFile" \
                -filter_complex \
                    "$filters,split=2 [a][b]; \
                    [a] palettegen=stats_mode=$stats_mode [pal]; \
                    [b] fifo [b]; \
                    [b] [pal] $palette_use" \
                -n "$out_file"

                notifyStatus $?

            elif [[ $start_time > 0 && $duration = 0 ]]
            then
                ffmpeg \
                -v warning \
                -ss $start_time \
                -i "$originalFile" \
                -filter_complex \
                    "$filters,split=2 [a][b]; \
                    [a] palettegen=stats_mode=$stats_mode [pal]; \
                    [b] fifo [b]; \
                    [b] [pal] $palette_use" \
                -n "$out_file"

                notifyStatus $?

            elif [[ $start_time > 0 && $duration > 0 ]]
            then
                ffmpeg \
                -v warning \
                -ss $start_time \
                -t $duration \
                -i "$originalFile" \
                -filter_complex \
                    "$filters,split=2 [a][b]; \
                    [a] palettegen=stats_mode=$stats_mode [pal]; \
                    [b] fifo [b]; \
                    [b] [pal] $palette_use" \
                -n "$out_file"

                notifyStatus $?

            fi
        fi

    else
        echo -e "$folder\n... is not vald"
        exit 1
    fi
fi
```
