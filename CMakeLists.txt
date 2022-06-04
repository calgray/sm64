
set(TARGET_NAME sm64plus)
set(LD_SCRIPT sm64.ld)
# replace /usr with N64SDK_DIR in .ld

# build ROM
set( BIN_NAME             ${TARGET_NAME}.bin )
set( ROM_NAME             ${TARGET_NAME}.z64 )
set( VERSION              us )
set( LOCALE               us )

# TODO remove
# Tested with Unix Makefiles build dir
set(BUILD_DIR             "CMakeFiles/${TARGET_NAME}.dir")
set(SM64_TARGET_DIR       "CMakeFiles/${TARGET_NAME}.dir")

# TODO remove
add_custom_target(PRE-${TARGET_NAME})

# file(GLOB_RECURSE C_SRCS RELATIVE ${CMAKE_CURRENT_LIST_DIR} src/goddard/*.c)
# add_library(goddard
#     ${C_SRCS}
# )

# original config is RELWITHDEBINFO
#file(GLOB_RECURSE SOUND_SRCS RELATIVE ${CMAKE_CURRENT_LIST_DIR} sound/*.c)
file(GLOB_RECURSE ASM_SRCS RELATIVE ${CMAKE_CURRENT_LIST_DIR}
    asm/*.s
    lib/rsp.s
    #assets/*.s
)
file(GLOB_RECURSE C_SRCS RELATIVE ${CMAKE_CURRENT_LIST_DIR}
    src/engine/*.c
    src/buffers/*.c
    src/audio/*.c
    src/menu/*.c
    src/game/*.c
    sound/*.c
    levels/*.c
    #actors/common0.c
    #actors/common1.c
    actors/*_geo.c
    data/*.c
)
list(FILTER C_SRCS EXCLUDE REGEX .inc.c)
list(FILTER C_SRCS EXCLUDE REGEX /eu/)

file(GLOB_RECURSE CPP_SRCS RELATIVE ${CMAKE_CURRENT_LIST_DIR} src/*.cpp)

set(TOOLS_DIR ${CMAKE_CURRENT_LIST_DIR}/tools)

set(bin_target_dir CMakeFiles/_bin.dir)

#########
# Start!
#########


#####################################
# preprocess localized text sources
#####################################
set(TEXT_SRCS
    text/define_courses.inc.c
    text/define_text.inc.c
)

# WARNING!!! custom output dir must start with either _,a,b,c,d,e,f,g,h,i,j,k,l,m,n,o
set(text_target_dir "CMakeFiles/_localized_text.dir")
add_custom_target(localized_text)
# TODO!!! WORKING_DIRECTORY ${text_target_dir})
foreach(TEXT_SRC ${TEXT_SRCS})
    cmake_path(GET TEXT_SRC PARENT_PATH file_dir)
    cmake_path(GET TEXT_SRC FILENAME file_out)
    set(out_dir ${file_dir}/${LOCALE})

    set(out_path ${out_dir}/${file_out}) # e.g. text/us/define_text.inc.c
    set(build_out_path ${text_target_dir}/${out_path})
    set(full_out_path ${CMAKE_CURRENT_BINARY_DIR}/${build_out_path})

    add_custom_command(
        OUTPUT ${full_out_path}.preprocessed
        MAIN_DEPENDENCY ${TEXT_SRC}
        COMMAND
            mkdir -p ${text_target_dir}/${out_dir}
        COMMAND
            ${CMAKE_C_PREPROCESSOR} -P -Wno-trigraphs
            -I${CMAKE_CURRENT_LIST_DIR}/include
            -I${BUILD_DIR}
            -I${BUILD_DIR}/include
            -I${CMAKE_CURRENT_LIST_DIR}/src
            -I${CMAKE_CURRENT_LIST_DIR}
            -I${CMAKE_CURRENT_LIST_DIR}/include/libc
            -I${CMAKE_CURRENT_LIST_DIR}/text/us
            -DVERSION_US=1 -DF3D_OLD=1
            -o ${full_out_path}.preprocessed
            ${CMAKE_CURRENT_LIST_DIR}/${TEXT_SRC}
    )
    add_custom_command(
        OUTPUT ${full_out_path}
        MAIN_DEPENDENCY ${full_out_path}.preprocessed
        BYPRODUCTS ${full_out_path}
        COMMENT "Generating ${build_out_path}"
        COMMAND
            ${TOOLS_DIR}/textconv ${CMAKE_CURRENT_LIST_DIR}/charmap.txt
            ${full_out_path}.preprocessed
            ${full_out_path}
    )
    target_sources(localized_text PRIVATE ${full_out_path})
endforeach()

##################################
# generate localized text headers
##################################
set(TEXT_HEADERS
    include/text_strings.h.in
    include/text_menu_strings.h.in
)
foreach(TEXT_HEADER ${TEXT_HEADERS})
    cmake_path(GET TEXT_HEADER PARENT_PATH file_dir)
    cmake_path(GET TEXT_HEADER STEM LAST_ONLY file_out)
    set(out_dir ${LOCALE}/${file_dir})

    set(out_path ${out_dir}/${file_out}) # e.g. us/include/text_strings.h
    set(build_out_path ${text_target_dir}/${out_path})
    set(full_out_path ${CMAKE_CURRENT_BINARY_DIR}/${build_out_path})

    add_custom_command(
        OUTPUT ${full_out_path}
        COMMENT "Generating ${build_out_path}"
        COMMAND mkdir -p ${text_target_dir}/${out_dir}
        COMMAND ${TOOLS_DIR}/textconv ${CMAKE_CURRENT_LIST_DIR}/charmap.txt
        ${CMAKE_CURRENT_LIST_DIR}/${TEXT_HEADER}
        ${full_out_path}
        BYPRODUCTS ${full_out_path}
        MAIN_DEPENDENCY ${CMAKE_CURRENT_LIST_DIR}/${TEXT_HEADER}
    )
    target_sources(localized_text PRIVATE ${full_out_path})
endforeach()

############################
# Preprocess Level Headers
############################
set(level_headers_target_dir "CMakeFiles/_levels.dir")
add_custom_target(level_headers)
set(LEVEL_HEADER
    levels/level_headers.h.in
)
cmake_path(GET LEVEL_HEADER PARENT_PATH file_dir)
cmake_path(GET LEVEL_HEADER STEM LAST_ONLY file_out)
set(out_dir ${LOCALE}/include)

set(out_path ${out_dir}/${file_out}) # e.g. us/include/level_headers.h
set(build_out_path ${level_headers_target_dir}/${out_path})
set(full_out_path ${CMAKE_CURRENT_BINARY_DIR}/${build_out_path})

add_custom_command(
    OUTPUT ${full_out_path}
    COMMENT "Generating ${build_out_path}"
    COMMAND mkdir -p ${level_headers_target_dir}/${out_dir}
    #COMMAND ${CMAKE_C_PREPROCESSOR} -Wno-trigraphs
    COMMAND clang -E -P -x c -Wno-trigraphs
    -I${CMAKE_CURRENT_LIST_DIR}/include
    -I${BUILD_DIR}
    -I${BUILD_DIR}/include
    -I${CMAKE_CURRENT_LIST_DIR}/src
    -I${CMAKE_CURRENT_LIST_DIR}
    -I${CMAKE_CURRENT_LIST_DIR}/include/libc
    -DVERSION_US=1 -DF3D_OLD=1
    ${CMAKE_CURRENT_LIST_DIR}/${LEVEL_HEADER} | sed -E 's|\(.+\)|\#include \"\\1\"|' > ${full_out_path}
    BYPRODUCTS ${full_out_path}
    MAIN_DEPENDENCY ${CMAKE_CURRENT_LIST_DIR}/${LEVEL_HEADER}
)
target_sources(level_headers PRIVATE ${full_out_path})

###########################
# generate texture sources
###########################
add_custom_target(textures)
set(textures_target_dir "CMakeFiles/_textures.dir")
set(TEXTURE_ENCODING u8)
file(GLOB_RECURSE IMAGE_SRCS RELATIVE ${CMAKE_CURRENT_LIST_DIR}
    textures/*.rgba32.png
    textures/*.rgba16.png
    textures/*.ia16.png
    textures/*.ia8.png
    textures/*.ia4.png
    textures/*.ia1.png
    textures/*.i8.png
    textures/*.i4.png
    levels/*.rgba32.png
    levels/*.rgba16.png
    levels/*.ia16.png
    levels/*.ia8.png
    levels/*.ia4.png
    levels/*.ia1.png
    levels/*.i8.png
    levels/*.i4.png
    actors/*.rgba32.png
    actors/*.rgba16.png
    actors/*.ia16.png
    actors/*.ia8.png
    actors/*.ia4.png
    actors/*.ia1.png
    actors/*.i8.png
    actors/*.i4.png
)
foreach(IMAGE_SRC ${IMAGE_SRCS})
    cmake_path(GET IMAGE_SRC PARENT_PATH file_dir)
    cmake_path(GET IMAGE_SRC STEM LAST_ONLY file_out)
    cmake_path(GET IMAGE_SRC EXTENSION file_ext)
    string(REGEX MATCHALL "\\.[^.]*" file_exts ${file_ext})
    list(REVERSE file_exts)
    list(GET file_exts 1 file_type)
    string(REGEX MATCH "[^.]*$" file_type ${file_type}) # e.g. rgba16
    # Use raw for fonts
    if(file_type STREQUAL "ia1")
        set(scheme raw)
        set(out_ext "")
    else()
        set(scheme ${TEXTURE_ENCODING})
        set(out_ext .inc.c)
    endif()
    
    set(out_dir ${LOCALE}/${file_dir})
    set(out_path ${out_dir}/${file_out}${out_ext}) # e.g. us/textures/cave/hmc_textures.00000.rgba16.inc.c
    set(build_out_path ${textures_target_dir}/${out_path})
    set(full_out_path ${CMAKE_CURRENT_BINARY_DIR}/${build_out_path})
    add_custom_command(
        OUTPUT ${full_out_path}
        COMMENT "Generating ${build_out_path}"
        COMMAND mkdir -p ${textures_target_dir}/${out_dir}
        COMMAND
        ${TOOLS_DIR}/n64graphics
        -s ${scheme}
        -i ${build_out_path}
        -g ${CMAKE_CURRENT_LIST_DIR}/${IMAGE_SRC}
        -f ${file_type}
        BYPRODUCTS ${full_out_path}
        MAIN_DEPENDENCY ${CMAKE_CURRENT_LIST_DIR}/${IMAGE_SRC}
    )
    target_sources(textures PRIVATE ${full_out_path})
endforeach()

################################
# generate image color indexes
###############################

# add_library(images OBJECT
#     ${IMAGE_SRCS}
# )
# add_dependencies(${TARGET_NAME} images)


######################
# generate skyboxes
######################
add_custom_target(sky)
#set(sky_target_dir "CMakeFiles/_sky.dir")
set(sky_target_dir ${bin_target_dir})

# Cake
set(CAKE_SRC levels/ending/cake.png)
set(out_dir ${LOCALE}/levels/ending)
set(out_path ${out_dir}/cake.inc.c)
set(build_out_path ${sky_target_dir}/${out_path})
set(full_out_path ${CMAKE_CURRENT_BINARY_DIR}/${build_out_path})
add_custom_command(
    OUTPUT ${full_out_path}
    COMMENT "Generating ${sky_target_dir}/${out_path}"
    COMMAND mkdir -p ${sky_target_dir}/${out_dir}
    COMMAND ${TOOLS_DIR}/skyconv
    --type cake
    --split ${CMAKE_CURRENT_LIST_DIR}/${CAKE_SRC}
    ${sky_target_dir}/${out_dir}
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
)
target_sources(sky PRIVATE ${full_out_path})

# skyboxes
set(SKY_SRCS
    textures/skyboxes/bbh.png
    textures/skyboxes/bidw.png
    textures/skyboxes/bitfs.png
    textures/skyboxes/bits.png
    textures/skyboxes/ccm.png
    textures/skyboxes/cloud_floor.png
    textures/skyboxes/clouds.png
    textures/skyboxes/ssl.png
    textures/skyboxes/water.png
    textures/skyboxes/wdw.png
)

foreach(SKY_SRC ${SKY_SRCS})
    cmake_path(GET SKY_SRC PARENT_PATH file_dir)
    cmake_path(GET SKY_SRC STEM LAST_ONLY file_out)
    set(out_dir ${LOCALE}/bin)
    #Split
    set(out_path ${out_dir}/${file_out}_skybox.c)
    set(build_out_path ${sky_target_dir}/${out_path})
    set(full_out_path ${CMAKE_CURRENT_BINARY_DIR}/${build_out_path})
    add_custom_command(
        OUTPUT ${full_out_path}
        COMMENT "Splitting ${SKY_SRC} -> ${sky_target_dir}/${out_path}"
        COMMAND mkdir -p ${sky_target_dir}/${out_dir}
        COMMAND ${TOOLS_DIR}/skyconv
        --type sky
        --split ${CMAKE_CURRENT_LIST_DIR}/${SKY_SRC}
        ${sky_target_dir}/${out_dir}
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    )

    # Compile and linking to ELF

    set(build_in_path ${full_out_path})
    set(out_path ${out_dir}/${file_out}_skybox.elf)
    set(build_out_path ${sky_target_dir}/${out_path})
    set(full_out_path ${CMAKE_CURRENT_BINARY_DIR}/${build_out_path})
    add_executable(${file_out}_skybox
        ${build_in_path}
    )
    set_target_properties(${file_out}_skybox PROPERTIES
        PREFIX ""
        OUTPUT_NAME ${file_out}_skybox
        SUFFIX ".elf"
        ARCHIVE_OUTPUT_DIRECTORY ${sky_target_dir}/${out_dir}
        LIBRARY_OUTPUT_DIRECTORY ${sky_target_dir}/${out_dir}
        RUNTIME_OUTPUT_DIRECTORY ${sky_target_dir}/${out_dir}
    )
    target_compile_options(${file_out}_skybox PRIVATE
        -nostdinc
        -mips3
        -mno-shared
        -mhard-float
        -mdivide-breaks
        -fno-stack-protector
        -fno-common
        -fno-zero-initialized-in-bss
        -fno-PIC
        -mno-abicalls
        -fno-strict-aliasing
        -fno-inline-functions
        -ffreestanding
        -fwrapv
        -Wall
        -Wextra
    )
    target_compile_definitions(${file_out}_skybox PRIVATE
        TARGET_N64=1
        _LANGUAGE_C=1
        VERSION_US=1
        F3D_OLD=1
        NON_MATCHING=1
        AVOID_UB=1
    )
    target_include_directories(${file_out}_skybox PRIVATE
        #.
        include
        include/libc
        #build/us
        #build/us/include
        #src
        # prebuild sources
        # ${CMAKE_CURRENT_BINARY_DIR}/${BUILD_DIR} # textures
        # ${CMAKE_CURRENT_BINARY_DIR}/${BUILD_DIR}/include # text
    )
    target_link_options(${file_out}_skybox PRIVATE
        -mabi=32
        -nostdlib
        -e 0
        -Ttext=0x0A000000
        -Wl,-Map,${sky_target_dir}/${out_dir}/${file_out}_skybox.elf.map
    )
    list(APPEND SEG_FILES ${full_out_path})

    # Extracting Compressible Data
    set(build_in_path ${sky_target_dir}/${out_dir}/${file_out}_skybox.elf)
    set(out_path ${out_dir}/${file_out}_skybox.bin)
    set(build_out_path ${sky_target_dir}/${out_path})
    set(full_out_path ${CMAKE_CURRENT_BINARY_DIR}/${build_out_path})
    add_custom_command(TARGET ${file_out}_skybox POST_BUILD
        #OUTPUT ${full_out_path}
        COMMENT "Extracting Compressible data to ${sky_target_dir}/${out_path}"
        COMMAND
            ${CMAKE_OBJCOPY}
            -O binary
            --only-section=.data
            ${build_in_path}
            ${build_out_path}
        BYPRODUCTS ${full_out_path}
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    )
    # Compressing
    set(build_in_path ${sky_target_dir}/${out_dir}/${file_out}_skybox.bin)
    set(out_path ${out_dir}/${file_out}_skybox.mio0)
    set(build_out_path ${sky_target_dir}/${out_path})
    set(full_out_path ${CMAKE_CURRENT_BINARY_DIR}/${build_out_path})
    add_custom_command(TARGET ${file_out}_skybox POST_BUILD
        COMMENT "Compressing to ${sky_target_dir}/${out_path}"
        COMMAND
            ${TOOLS_DIR}/mio0
            ${build_in_path}
            ${build_out_path}
        BYPRODUCTS ${full_out_path}
        MAIN_DEPENDENCY ${CMAKE_CURRENT_BINARY_DIR}/${build_in_path}
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    )
    # Converting MIO0 to ELF
    set(build_in_path ${sky_target_dir}/${out_dir}/${file_out}_skybox.mio0)
    set(out_path ${out_dir}/${file_out}_skybox.mio0.o)
    set(build_out_path ${sky_target_dir}/${out_path})
    set(full_out_path ${CMAKE_CURRENT_BINARY_DIR}/${build_out_path})
    add_custom_command(TARGET ${file_out}_skybox POST_BUILD
        COMMENT "Converting to ${sky_target_dir}/${out_path}"
        COMMAND
            mips64-elf-ld -r -b binary
            ${build_in_path} 
            -o ${build_out_path}
        BYPRODUCTS ${full_out_path}
        MAIN_DEPENDENCY ${CMAKE_CURRENT_BINARY_DIR}/${build_in_path}
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    )

    #target_sources(sky PRIVATE ${full_out_path})
    add_dependencies(sky ${file_out}_skybox)
endforeach()

#####################
# Generate sounds
#####################

set(sound_target_dir CMakeFiles/_sound.dir)
add_custom_target(sound)
file(GLOB_RECURSE SOUND_SAMPLES RELATIVE ${CMAKE_CURRENT_LIST_DIR} sound/*.aiff)
foreach(SOUND_SAMPLE ${SOUND_SAMPLES})
    cmake_path(GET SOUND_SAMPLE PARENT_PATH file_dir)
    cmake_path(GET SOUND_SAMPLE STEM LAST_ONLY file_out)
    set(out_dir ${LOCALE}/${file_dir})
    # Extract table
    set(out_path ${out_dir}/${file_out}.table)
    set(build_out_path ${sound_target_dir}/${out_path})
    set(full_out_path ${CMAKE_CURRENT_BINARY_DIR}/${build_out_path})
    add_custom_command(
        OUTPUT ${full_out_path}
        COMMENT "Extracting to ${sound_target_dir}/${out_path}"
        COMMAND mkdir -p ${sound_target_dir}/${out_dir}
        COMMAND ${TOOLS_DIR}/aiff_extract_codebook
        ${CMAKE_CURRENT_LIST_DIR}/${SOUND_SAMPLE}
        > ${build_out_path}
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    )
    # Encode ADPCM
    set(build_in_path ${full_out_path})
    set(out_path ${out_dir}/${file_out}.aifc)
    set(build_out_path ${sound_target_dir}/${out_path})
    set(full_out_path ${CMAKE_CURRENT_BINARY_DIR}/${build_out_path})
    add_custom_command(
        OUTPUT ${full_out_path}
        MAIN_DEPENDENCY ${build_in_path}
        COMMENT "Encoding ADPCM to ${sound_target_dir}/${out_path}"
        COMMAND ${TOOLS_DIR}/vadpcm_enc
        -c ${build_in_path}
        ${CMAKE_CURRENT_LIST_DIR}/${SOUND_SAMPLE}
        ${build_out_path}
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    )
    target_sources(sound PRIVATE ${full_out_path})
endforeach()

#endian-bitwidth flags
set(sound_flags_target endian-and-bitwidth)
set(sound_defs_path ${CMAKE_CURRENT_BINARY_DIR}/${sound_target_dir}/${LOCALE}/${sound_flags_target})
add_custom_command(
    OUTPUT ${sound_defs_path}.txt
    COMMAND mkdir -p ${CMAKE_CURRENT_BINARY_DIR}/${sound_target_dir}/${LOCALE}
    COMMAND 
        ${CMAKE_C_COMPILER}
        -Iinclude
        ${TOOLS_DIR}/determine-endian-bitwidth.c
        2>${sound_defs_path}.dummy1 || true
    COMMAND
        grep -o 'msgbegin --endian .* --bitwidth .* msgend'
        ${sound_defs_path}.dummy1 > ${sound_defs_path}.dummy2
    COMMAND
        head -n1 <${sound_defs_path}.dummy2 | cut -d' ' -f2-5 > ${sound_defs_path}.txt
    WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}
)

#generate CTL
set(sound_data_dir ${CMAKE_CURRENT_BINARY_DIR}/${sound_target_dir}/${LOCALE}/sound)
add_custom_command(
    OUTPUT ${sound_data_dir}/sound_data.ctl ${sound_data_dir}/sound_data.tbl
    MAIN_DEPENDENCY ${CMAKE_CURRENT_BINARY_DIR}/${sound_target_dir}/${LOCALE}/${sound_flags_target}.txt
    COMMAND
        python3 ${TOOLS_DIR}/assemble_sound.py 
        ${sound_data_dir}/samples/
        sound/sound_banks/
        ${sound_data_dir}/sound_data.ctl
        ${sound_data_dir}/ctl_header
        ${sound_data_dir}/sound_data.tbl
        ${sound_data_dir}/tbl_header
        -DVERSION_US=1
        -DF3D_OLD=1
        -DNON_MATCHING=1
        -DAVOID_UB=1
        $$\(cat ${CMAKE_CURRENT_BINARY_DIR}/${sound_target_dir}/${LOCALE}/${sound_flags_target}.txt\)
    WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}
)

# generate CTL inc.c
add_custom_command(
    OUTPUT ${sound_data_dir}/sound_data.ctl.inc.c
    MAIN_DEPENDENCY ${sound_data_dir}/sound_data.ctl
    COMMAND
    hexdump -v -e '1/1 \"0x%X,\"' ${sound_data_dir}/sound_data.ctl > ${sound_data_dir}/sound_data.ctl.inc.c
)
target_sources(sound PRIVATE ${sound_data_dir}/sound_data.ctl.inc.c)

# generate TBL inc.c
add_custom_command(
    OUTPUT ${sound_data_dir}/sound_data.tbl.inc.c
    MAIN_DEPENDENCY ${sound_data_dir}/sound_data.tbl
    COMMAND
    hexdump -v -e '1/1 \"0x%X,\"' ${sound_data_dir}/sound_data.tbl > ${sound_data_dir}/sound_data.tbl.inc.c
)
target_sources(sound PRIVATE ${sound_data_dir}/sound_data.tbl.inc.c)

# generate sound_player.m64
add_custom_command(
    OUTPUT ${sound_data_dir}/sequences/00_sound_player.o
    MAIN_DEPENDENCY sound/sequences/00_sound_player.s
    COMMAND mkdir -p ${sound_data_dir}/sequences
    COMMAND
        ${CMAKE_C_PREPROCESSOR} -P -Wno-trigraphs
        -Iinclude #-Isrc #-I. #-Iinclude/libc
        -DVERSION_US=1 -DF3D_OLD=1 -DNON_MATCHING=1 -DAVOID_UB=1
        sound/sequences/00_sound_player.s
        | ${CMAKE_C_COMPILER}
        -march=vr4300 -mabi=32
        -Iinclude -Ibuild/us -Ibuild/us/include -Isrc -I. -Iinclude/libc
        -DVERSION_US=1 -DF3D_OLD=1 -DNON_MATCHING=1 -DAVOID_UB=1
        -x assembler-with-cpp
        -MMD -MF ${sound_data_dir}/sequences/00_sound_player.d
        -o ${sound_data_dir}/sequences/00_sound_player.o
        -
    WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}
)
add_custom_command(
    OUTPUT ${sound_data_dir}/sequences/00_sound_player.m64
    MAIN_DEPENDENCY ${sound_data_dir}/sequences/00_sound_player.o
    COMMAND
        ${CMAKE_OBJCOPY} -j .rodata
        ${sound_data_dir}/sequences/00_sound_player.o
        -O binary ${sound_data_dir}/sequences/00_sound_player.m64
    WORKING_DIRECTORY ${CMAKE_CURRENT_BIN_DIR}
)
#target_sources(sound PRIVATE ${sound_data_dir}/sequences/00_sound_player.m64)

# generate sequences.bin
file(GLOB_RECURSE SOUND_SEQUENCES RELATIVE ${CMAKE_CURRENT_LIST_DIR} sound/sequences/${LOCALE}/*.m64)
add_custom_command(
    OUTPUT ${sound_data_dir}/sequences.bin ${sound_data_dir}/bank_sets
    MAIN_DEPENDENCY ${sound_data_dir}/sequences/00_sound_player.m64
    COMMAND
        python ${TOOLS_DIR}/assemble_sound.py
        --sequences ${sound_data_dir}/sequences.bin
        ${sound_data_dir}/sequences_header
        ${sound_data_dir}/bank_sets
        sound/sound_banks/
        sound/sequences.json
        ${sound_data_dir}/sequences/00_sound_player.m64
        ${SOUND_SEQUENCES}
        -DVERSION_US=1
        -DF3D_OLD=1
        -DNON_MATCHING=1
        -DAVOID_UB=1
        $$\(cat ${CMAKE_CURRENT_BINARY_DIR}/${sound_target_dir}/${LOCALE}/${sound_flags_target}.txt\)
    WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}
)

# generate BIN inc.c
add_custom_command(
    OUTPUT ${sound_data_dir}/sequences.bin.inc.c
    MAIN_DEPENDENCY ${sound_data_dir}/sequences.bin
    COMMAND
    hexdump -v -e '1/1 \"0x%X,\"' ${sound_data_dir}/sequences.bin > ${sound_data_dir}/sequences.bin.inc.c
)
target_sources(sound PRIVATE ${sound_data_dir}/sequences.bin.inc.c)

# generate bank_sets inc.c
add_custom_command(
    OUTPUT ${sound_data_dir}/bank_sets.inc.c
    MAIN_DEPENDENCY ${sound_data_dir}/bank_sets
    COMMAND
    hexdump -v -e '1/1 \"0x%X,\"' ${sound_data_dir}/bank_sets > ${sound_data_dir}/bank_sets.inc.c
)
target_sources(sound PRIVATE ${sound_data_dir}/bank_sets.inc.c)

######################
# Generate animations
######################

set(animation_target animation)
add_library(${animation_target} OBJECT)
set(animation_target_dir CMakeFiles/animation.dir)
#mario animation data
set(full_out_path ${CMAKE_CURRENT_BINARY_DIR}/${LOCALE}/assets/mario_anim_data.c)
add_custom_command(
    OUTPUT ${full_out_path}
    COMMAND mkdir -p ${CMAKE_CURRENT_BINARY_DIR}/${LOCALE}/assets
    COMMAND python3 ${TOOLS_DIR}/mario_anims_converter.py > ${full_out_path}
    WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}
)
target_sources(${animation_target} PRIVATE ${full_out_path})
# demo data
set(full_out_path ${CMAKE_CURRENT_BINARY_DIR}/${LOCALE}/assets/demo_data.c)
add_custom_command(
    OUTPUT ${full_out_path}
    COMMAND mkdir -p ${CMAKE_CURRENT_BINARY_DIR}/${LOCALE}/assets
    COMMAND python3 ${TOOLS_DIR}/demo_data_converter.py
    assets/demo_data.json
    -Iinclude
    -Ibuild/us
    -Ibuild/us/include
    -Isrc
    -I.
    -Iinclude/libc
    -DVERSION_US=1
    -DF3D_OLD=1
    -DNON_MATCHING=1
    -DAVOID_UB=1 > build/us/assets/demo_data.c > ${full_out_path}
    WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}
)
target_sources(${animation_target} PRIVATE ${full_out_path})
# compile
# set_target_properties(${bin_target} PROPERTIES
#     PREFIX ""
#     OUTPUT_NAME ${file_out}
#     SUFFIX ".elf"
#     ARCHIVE_OUTPUT_DIRECTORY ${common_target_dir}/${out_dir}
#     LIBRARY_OUTPUT_DIRECTORY ${common_target_dir}/${out_dir}
#     RUNTIME_OUTPUT_DIRECTORY ${common_target_dir}/${out_dir}
# )
target_compile_definitions(${animation_target} PRIVATE
    TARGET_N64=1
    _LANGUAGE_C=1
    VERSION_US=1
    F3D_OLD=1
    NON_MATCHING=1
    AVOID_UB=1
)
target_compile_options(${animation_target} PRIVATE
    -nostdinc
    -mips3
    -mno-shared
    -mhard-float
    -mdivide-breaks
    -fno-stack-protector
    -fno-common
    -fno-zero-initialized-in-bss
    -fno-PIC
    -mno-abicalls
    -fno-strict-aliasing
    -fno-inline-functions
    -ffreestanding
    -fwrapv
    -Wall
    -Wextra
)
target_include_directories(${animation_target} PRIVATE
    include
    include/libc
    src
    .
    ${CMAKE_CURRENT_BINARY_DIR}/${textures_target_dir}/${LOCALE}
    ${CMAKE_CURRENT_BINARY_DIR}/${text_target_dir}
)
target_link_libraries(${animation_target} PRIVATE n64)


########################
# generate rsp binaries
########################
add_custom_target(rsp)
set(rsp_target_dir CMakeFiles/_rsp.dir)
file(GLOB_RECURSE RSP_SRCS RELATIVE ${CMAKE_CURRENT_LIST_DIR} rsp/*.s)
foreach(RSP_SRC ${RSP_SRCS})
    cmake_path(GET RSP_SRC PARENT_PATH file_dir)
    cmake_path(GET RSP_SRC STEM LAST_ONLY file_out)
    set(out_dir ${LOCALE}/${file_dir})
    set(out_path ${out_dir}/${file_out}.bin)
    set(build_out_path ${rsp_target_dir}/${out_path})
    set(full_out_path ${CMAKE_CURRENT_BINARY_DIR}/${build_out_path})
    set(full_data_out_path ${CMAKE_CURRENT_BINARY_DIR}/${rsp_target_dir}/${out_dir}/${file_out}_data.bin)

    add_custom_command(
        OUTPUT ${full_out_path}
        COMMENT "Generating rsp file ${file_dir}/${file_out}.bin"
        COMMAND mkdir -p ${rsp_target_dir}/${out_dir}
        COMMAND
        ${TOOLS_DIR}/armips
        -sym ${full_out_path}.sym
        -definelabel VERSION_US 1
        -definelabel F3D_OLD 1
        -root ${CMAKE_CURRENT_LIST_DIR}
        -strequ CODE_FILE ${full_out_path}
        -strequ DATA_FILE ${full_data_out_path}
        ${CMAKE_CURRENT_LIST_DIR}/${RSP_SRC}
        BYPRODUCTS
        ${full_out_path}
        ${full_out_path}.sym
        ${full_data_out_path}
        MAIN_DEPENDENCY ${CMAKE_CURRENT_LIST_DIR}/${RSP_SRC}
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    )
    target_sources(rsp PRIVATE ${full_out_path} ${full_data_out_path})
endforeach()


#############################################################
# Common (bin) Textures + Actors Compressed ELF Load Points
#############################################################
add_custom_target(common)
#set(common_target_dir "CMakeFiles/_common.dir")
set(common_target_dir ${bin_target_dir})
file(GLOB_RECURSE COMMON_SRCS RELATIVE ${CMAKE_CURRENT_LIST_DIR}
    bin/*.c
    actors/group0.c
    actors/group1.c
    actors/group2.c
    actors/group3.c
    actors/group4.c
    actors/group5.c
    actors/group6.c
    actors/group7.c
    actors/group8.c
    actors/group9.c
    actors/group10.c
    actors/group11.c
    actors/group12.c
    actors/group13.c
    actors/group14.c
    actors/group15.c
    actors/group16.c
    actors/group17.c
    actors/common0.c
    actors/common1.c
)

#exclude actors model.inc.c
list(FILTER COMMON_SRCS EXCLUDE REGEX .inc.c)
list(FILTER COMMON_SRCS EXCLUDE REGEX /eu/)

foreach(COMMON_SRC ${COMMON_SRCS})
    cmake_path(GET COMMON_SRC PARENT_PATH file_dir)
    cmake_path(GET COMMON_SRC STEM file_out)
    # string(REGEX MATCH "(.*)\\.[^.]+$" bin_target ${COMMON_SRC})
    # string(REGEX REPLACE "/" "." bin_target ${COMMON_SRC})

    set(bin_target bin_${file_out})
    set(out_dir ${LOCALE}/${file_dir})
    set(out_path ${out_dir}/${file_out}.elf)
    set(build_out_path ${common_target_dir}/${out_path})
    set(full_out_path ${CMAKE_CURRENT_BINARY_DIR}/${build_out_path})

    # add_custom_target(${bin_target}_dir
    #     COMMAND mkdir -p ${CMAKE_CURRENT_BINARY_DIR}/${common_target_dir}/${out_dir}
    # )
    add_executable(${bin_target}
        ${COMMON_SRC}
    )
    #TODO only using sky here to create common_target_dir
    add_dependencies(${bin_target} localized_text textures) # ${bin_target}_dir)
    set_target_properties(${bin_target} PROPERTIES
        PREFIX ""
        OUTPUT_NAME ${file_out}
        SUFFIX ".elf"
        ARCHIVE_OUTPUT_DIRECTORY ${common_target_dir}/${out_dir}
        LIBRARY_OUTPUT_DIRECTORY ${common_target_dir}/${out_dir}
        RUNTIME_OUTPUT_DIRECTORY ${common_target_dir}/${out_dir}
    )
    target_compile_definitions(${bin_target} PRIVATE
        TARGET_N64=1
        _LANGUAGE_C=1
        VERSION_US=1
        F3D_OLD=1
        NON_MATCHING=1
        AVOID_UB=1
    )
    target_compile_options(${bin_target} PRIVATE
        -nostdinc
        -mips3
        -mno-shared
        -mhard-float
        -mdivide-breaks
        -fno-stack-protector
        -fno-common
        -fno-zero-initialized-in-bss
        -fno-PIC
        -mno-abicalls
        -fno-strict-aliasing
        -fno-inline-functions
        -ffreestanding
        -fwrapv
        -Wall
        -Wextra
    )
    target_include_directories(${bin_target} PRIVATE
        include
        include/libc
        src
        .
        ${CMAKE_CURRENT_BINARY_DIR}/${textures_target_dir}/${LOCALE}
        ${CMAKE_CURRENT_BINARY_DIR}/${text_target_dir}
    )
    target_link_libraries(${bin_target} PRIVATE n64)
    target_link_options(${bin_target} PRIVATE
        -mabi=32
        -nostdlib
        -e 0
        -Ttext=0x09000000
        -Wl,-Map,${full_out_path}.map
    )
    list(APPEND SEG_FILES ${full_out_path})
    
    # extract compressible data
    add_custom_command(TARGET ${bin_target}
        POST_BUILD
        COMMENT "Extracting to ${file_out}.bin"
        COMMAND ${CMAKE_OBJCOPY} -O binary --only-section=.data
        ${full_out_path}
        ${CMAKE_CURRENT_BINARY_DIR}/${common_target_dir}/${out_dir}/${file_out}.bin
    )
    # compress
    add_custom_command(TARGET ${bin_target}
        POST_BUILD
        COMMENT "Compressing to ${file_out}.mio0"
        COMMAND ${TOOLS_DIR}/mio0
        ${CMAKE_CURRENT_BINARY_DIR}/${common_target_dir}/${out_dir}/${file_out}.bin
        ${CMAKE_CURRENT_BINARY_DIR}/${common_target_dir}/${out_dir}/${file_out}.mio0
    )
    # convert MIO00 to ELF.o
    add_custom_command(TARGET ${bin_target}
        POST_BUILD
        COMMENT "Converting to ELF ${file_out}.mio0.o"
        COMMAND
            mips64-elf-ld
            -r -b binary
            ${CMAKE_CURRENT_BINARY_DIR}/${common_target_dir}/${out_dir}/${file_out}.mio0
            -o ${CMAKE_CURRENT_BINARY_DIR}/${common_target_dir}/${out_dir}/${file_out}.mio0.o
    )
    add_dependencies(common ${bin_target})
endforeach()

############################################
# Generate Level Compressed ELF Load Points
############################################
add_custom_target(levels)
#set(levels_target_dir CMakeFiles/_levels.dir)
set(levels_target_dir ${bin_target_dir})

file(GLOB_RECURSE LEVEL_SRCS RELATIVE ${CMAKE_CURRENT_LIST_DIR}
    levels/*leveldata.c
)
foreach(LEVEL_SRC ${LEVEL_SRCS})
    cmake_path(GET LEVEL_SRC PARENT_PATH file_dir)
    cmake_path(GET LEVEL_SRC STEM file_out)
    
    string(REGEX REPLACE "/" "." level_target ${file_dir})
    set(level_name ${level_target})
    #cmake_path( level_target EXTENSION  level_name)

    set(out_dir ${LOCALE}/${file_dir})
    set(out_path ${out_dir}/${file_out}.elf)
    set(build_out_path ${levels_target_dir}/${out_path})
    set(full_out_path ${CMAKE_CURRENT_BINARY_DIR}/${build_out_path})

    add_executable(${level_target}
        ${LEVEL_SRC}
    )
    add_dependencies(${level_target} level_headers)
    add_dependencies(${level_target} sky)
    add_dependencies(${level_target} common)
    set_target_properties(${level_target} PROPERTIES
        PREFIX ""
        OUTPUT_NAME ${file_out}
        SUFFIX ".elf"
        ARCHIVE_OUTPUT_DIRECTORY ${levels_target_dir}/${out_dir}
        LIBRARY_OUTPUT_DIRECTORY ${levels_target_dir}/${out_dir}
        RUNTIME_OUTPUT_DIRECTORY ${levels_target_dir}/${out_dir}
    )
    target_compile_definitions(${level_target} PRIVATE
        TARGET_N64=1
        _LANGUAGE_C=1
        VERSION_US=1
        F3D_OLD=1
        NON_MATCHING=1
        AVOID_UB=1
    )
    target_compile_options(${level_target} PRIVATE
        -nostdinc
        -mips3
        -mno-shared
        -mhard-float
        -mdivide-breaks
        -fno-stack-protector
        -fno-common
        -fno-zero-initialized-in-bss
        -fno-PIC
        -mno-abicalls
        -fno-strict-aliasing
        -fno-inline-functions
        -ffreestanding
        -fwrapv
        -Wall
        -Wextra
    )
    target_include_directories(${level_target} PRIVATE
        include
        ${CMAKE_CURRENT_BINARY_DIR}/${textures_target_dir}/${LOCALE}
        ${CMAKE_CURRENT_BINARY_DIR}/${sky_target_dir}/${LOCALE}
        include/libc
        src
        .
    )
    target_link_libraries(${level_target} PRIVATE n64)
    
    if(EXISTS ${CMAKE_CURRENT_LIST_DIR}/${file_dir}/level.yaml)
    execute_process(
        COMMAND yq -r ".\"texture-bin\"" ${CMAKE_CURRENT_LIST_DIR}/${file_dir}/level.yaml
        OUTPUT_VARIABLE texture_bin
        OUTPUT_STRIP_TRAILING_WHITESPACE
    )
    execute_process(
        COMMAND yq -r ".\"texture-address\"" ${CMAKE_CURRENT_LIST_DIR}/${file_dir}/level.yaml
        OUTPUT_VARIABLE texture_address
        OUTPUT_STRIP_TRAILING_WHITESPACE
    )    
    target_link_options(${level_target} PRIVATE
        -mabi=32
        -nostdlib
        -e 0
        -Ttext=${texture_address}
        -Wl,-Map,${level_target}.elf.map,-R,${common_target_dir}/${LOCALE}/bin/${texture_bin}.elf
    )
    endif()
    list(APPEND SEG_FILES ${full_out_path})

    # extract compressible data
    add_custom_command(TARGET ${level_target}
        POST_BUILD
        COMMENT "Extracting to ${file_dir}/${file_out}.bin"
        COMMAND ${CMAKE_OBJCOPY} -O binary --only-section=.data
        ${full_out_path}
        ${CMAKE_CURRENT_BINARY_DIR}/${levels_target_dir}/${out_dir}/${file_out}.bin
    )
    # compress
    add_custom_command(TARGET ${level_target}
        POST_BUILD
        COMMENT "Compressing to ${file_dir}/${file_out}.mio0"
        COMMAND ${TOOLS_DIR}/mio0
        ${CMAKE_CURRENT_BINARY_DIR}/${levels_target_dir}/${out_dir}/${file_out}.bin
        ${CMAKE_CURRENT_BINARY_DIR}/${levels_target_dir}/${out_dir}/${file_out}.mio0
    )
    # convert MIO00 to ELF.o
    add_custom_command(TARGET ${level_target}
        POST_BUILD
        COMMENT "Converting to ELF ${file_dir}/${file_out}.mio0.o"
        COMMAND mips64-elf-ld -r -b binary
        ${CMAKE_CURRENT_BINARY_DIR}/${levels_target_dir}/${out_dir}/${file_out}.mio0
        -o ${CMAKE_CURRENT_BINARY_DIR}/${levels_target_dir}/${out_dir}/${file_out}.mio0.o
    )
    add_dependencies(levels ${level_target})
endforeach()



###########################
# Generate final project
###########################
add_executable(${TARGET_NAME}
    ${ASM_SRCS}
    ${C_SRCS}
    ${CPP_SRCS}
)
add_dependencies(${TARGET_NAME}
    localized_text
    textures
    sky
    sound
    rsp
    level_headers
    animation
    common
    levels
)

set_target_properties(${TARGET_NAME} PROPERTIES
    OUTPUT_NAME "${TARGET_NAME}"
    SUFFIX ".elf"
)
target_link_libraries(${TARGET_NAME} PRIVATE n64)

target_include_directories(${TARGET_NAME} PRIVATE .)
target_include_directories(${TARGET_NAME} PUBLIC
    include
    include/libc
    src
    #sound
    #assets
    ${N64SDK_DIR}/include/n64
    ${N64SDK_DIR}/include/n64/PR
    #${N64SDK_DIR}/include/n64/nusys
    ${CMAKE_SOURCE_DIR}/projects/nucpp/include

    # preprocessed headers
    ${CMAKE_CURRENT_BINARY_DIR}/${level_headers_target_dir}/${LOCALE}/include # level headers
    ${CMAKE_CURRENT_BINARY_DIR}/${text_target_dir}/${LOCALE}/include # text

    # preprocessed sources
    ${CMAKE_CURRENT_BINARY_DIR}/${textures_target_dir}/${LOCALE} # textures
    ${CMAKE_CURRENT_BINARY_DIR}/${sky_target_dir}/${LOCALE} # sky+cake
    ${CMAKE_CURRENT_BINARY_DIR}/${sound_target_dir}/${LOCALE} # sound
    ${CMAKE_CURRENT_BINARY_DIR}/${rsp_target_dir}/${LOCALE} # rsp
    ${CMAKE_CURRENT_BINARY_DIR}/${common_target_dir}/${LOCALE} # common
)
target_compile_definitions(${TARGET_NAME} PUBLIC 
    #_FINALROM=1
    #F3DEX_GBI_2=1
    #N_AUDIO=1
    # MAXWIDTH=320
    # MAXHEIGHT=240
    # _FORTIFY_SOURCE=2
    # NON_MATCHING=1
    # AVOID_UB=1
    # VERSION_US=1


    TARGET_N64=1
    _LANGUAGE_C=1
    VERSION_US=1
    F3D_OLD=1
    NON_MATCHING=1
    AVOID_UB=1
)

target_compile_options(${TARGET_NAME} PUBLIC
    -nostdinc
    -mips3
    -mno-shared
    -mhard-float
    -mdivide-breaks
    -fno-stack-protector
    -fno-common
    -fno-zero-initialized-in-bss
    -fno-PIC
    -mno-abicalls
    -fno-strict-aliasing
    -fno-inline-functions
    -ffreestanding
    -fwrapv
    -Wall
    -Wextra
)

###################
# build boot object
###################
# add_custom_command(TARGET ${TARGET_NAME}
#     PRE_BUILD
#     COMMAND ${CMAKE_OBJCOPY}
#     -I binary
#     -B mips
#     -O elf32-bigmips
#     ${BOOT_SRC}
#     ${BUILD_DIR}/${BOOT_OBJ}
#     COMMENT "generating boot object ${BUILD_DIR}/${BOOT_OBJ}"
#     MAIN_DEPENDENCY ${BOOT_SRC}
#     BYPRODUCTS ${BUILD_DIR}/${BOOT_OBJ}
# )


#####################
# build linker script with gcc preprocessor
#####################
set(MOD_FLAGS
    -D_FINALROM=1
    -DF3DEX_GBI_2=1
    -DNAUDIO=1
    -DBUILD_DIR=${BUILD_DIR}
    -DLIB_DIR=../../../../../projects/sm64plus/build/${LOCALE}
    -DSM64_TARGET_DIR=${SM64_TARGET_DIR}
    -DBIN_TARGET_DIR=${bin_target_dir}/${LOCALE}
    -DANIM_TARGET_DIR=${animation_target_dir}/${LOCALE}
    #-DLEVELS_TARGET_DIR=${levels_target_dir}/${LOCALE}
    # TODO!!!
    # linker script expects mio0.o for levels, skybox, and common in the same dir
    -DN64SDK_DIR=${N64SDK_DIR}
    -MMD -MP
)
set(MOD_INCLUDES
    -I${N64SDK_DIR}/include/n64
    -I${N64SDK_DIR}/include/n64/PR
    -I${N64SDK_DIR}/include/n64/nusys
    -I${CMAKE_CURRENT_BINARY_DIR}
    -I${CMAKE_CURRENT_BINARY_DIR}/include
    -I${CMAKE_CURRENT_LIST_DIR}
    -I${CMAKE_CURRENT_LIST_DIR}/include
    -Isrc
    -Isound
    -Iassets
    -I.
)
add_custom_command(TARGET ${TARGET_NAME}
    PRE_LINK
    COMMAND ${CMAKE_C_PREPROCESSOR} -P -Wno-trigraphs
        -undef
        ${MOD_INCLUDES}
        ${MOD_FLAGS}
        -MT ${BUILD_DIR}/${LD_SCRIPT}
        -MF ${BUILD_DIR}/${LD_SCRIPT}.d
        -o ${BUILD_DIR}/${LD_SCRIPT}
        ${CMAKE_CURRENT_LIST_DIR}/${LD_SCRIPT}
    COMMENT "Generating linker script ${LD_SCRIPT}"
    MAIN_DEPENDENCY ${CMAKE_CURRENT_LIST_DIR}/${LD_SCRIPT}
    DEPENDS ${BUILD_DIR}/${BOOT_OBJ}
    BYPRODUCTS ${BUILD_DIR}/${LD_SCRIPT} ${BUILD_DIR}/${LD_SCRIPT}.d
)

# TODO: library hack!
add_library(sm64plus::goddard STATIC IMPORTED)
set_target_properties(sm64plus::goddard
    PROPERTIES
    IMPORTED_LOCATION ${CMAKE_CURRENT_LIST_DIR}/build/${LOCALE}/libgoddard.a
    INTERFACE_INCLUDE_DIRECTORIES ${CMAKE_CURRENT_LIST_DIR}/src/goddard
)

add_library(sm64plus::ultra STATIC IMPORTED)
set_target_properties(sm64plus::ultra
    PROPERTIES
    IMPORTED_LOCATION ${CMAKE_CURRENT_LIST_DIR}/build/${LOCALE}/libultra.a
    #INTERFACE_INCLUDE_DIRECTORIES ${CMAKE_CURRENT_LIST_DIR}/include
)

###########
# LINKING
###########
list(TRANSFORM SEG_FILES PREPEND "-R,")
list(JOIN SEG_FILES , GCC_LDFLAGS)
target_link_options(${TARGET_NAME} PRIVATE
    -mabi=32
    -nostdlib
    -L ${BUILD_DIR}
    # ${BUILD_DIR}/${BOOT_OBJ}
    -Wl,-T,undefined_syms.txt,-T,${BUILD_DIR}/${LD_SCRIPT},-Map,${BUILD_DIR}/${TARGET_NAME}.map,--no-check-sections,${GCC_LDFLAGS}
)
target_link_directories(${TARGET_NAME}
    PUBLIC
    .
    ${BUILD_DIR}
    ${N64SDK_DIR}
)
# order specific
target_link_libraries(${TARGET_NAME} PRIVATE
    #n64sdkmod::nualstl
    #n64sdkmod::mus
    #n64sdkmod::nualstl_n
    #n64sdkmod::n_mus
    #n64sdkmod::n_audio_sc
    # n64sdkmod::nualsgi_n
    #n64sdkmod::hvq
    #n64sdkmod::nustd
    #n64sdkmod::nusys
    # n64sdkmod::nusys_d
    # n64sdkmod::n_audio
    
    #n64sdkmod::ultra_rom # TODO: remove -isystem
    sm64plus::goddard
    sm64plus::ultra
    # TODO!!! .ld needs to know exact goddard and ultra location
    #gcc
)

######
# Pad
######
add_custom_command(TARGET ${TARGET_NAME}
    POST_BUILD
    COMMAND ${CMAKE_OBJCOPY} --pad-to=0x800000 --gap-fill=0xFF $<TARGET_FILE:${TARGET_NAME}> ${BIN_NAME} -O binary
    COMMAND ${TOOLS_DIR}/n64cksum ${BIN_NAME} ${ROM_NAME}
    #COMMAND makemask ${ROM_NAME}
    #COMMAND mips64-elf-size -A -d ${TARGET_NAME}.elf
    #COMMAND stat --format='%n: %s bytes' * | grep ${TARGET_NAME}.elf
    BYPRODUCTS ${ROM_NAME} ${BIN_NAME}
    COMMENT "Generating ${ROM_NAME}"
)

#################
# Common Failures
#################
# undefined reference to `NU_SPEC_BOOT_STACK`
# - missing "-x assembler-with-cpp" on assembling asm objects

# defined in discarded section .rodata*
# - 