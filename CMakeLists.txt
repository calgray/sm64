
set(TARGET_NAME sm64plus)
set(LD_SCRIPT sm64.ld)
# replace /usr with N64SDK_DIR in .ld

# build ROM
#set( BIN_NAME             ${TARGET_NAME}.bin )
set( ROM_NAME             ${TARGET_NAME}.z64 )
set( VERSION              us )
set( LOCALE               us )

# Tested with Unix Makefiles build dir
set(BUILD_DIR             "CMakeFiles/${TARGET_NAME}.dir")

add_custom_target(PRE-${TARGET_NAME})
# add_library(PRE-${TARGET_NAME}
#     lib/rsp.s
# )

# file(GLOB_RECURSE C_SRCS RELATIVE ${CMAKE_CURRENT_LIST_DIR} src/goddard/*.c)
# add_library(goddard
#     ${C_SRCS}
# )

# original config is RELWITHDEBINFO
file(GLOB_RECURSE ASM_SRCS RELATIVE ${CMAKE_CURRENT_LIST_DIR} asm/*.s lib/*.s)
file(GLOB_RECURSE SOUND_SRCS RELATIVE ${CMAKE_CURRENT_LIST_DIR} sound/*.s)
file(GLOB_RECURSE ASSETS_SRCS RELATIVE ${CMAKE_CURRENT_LIST_DIR} assets/*.s)

file(GLOB_RECURSE C_SRCS RELATIVE ${CMAKE_CURRENT_LIST_DIR}
    src/*.c
    levels/*.c
    actors/*_geo.c
    data/*.c
)
list(FILTER C_SRCS EXCLUDE REGEX .inc.c)
list(FILTER C_SRCS EXCLUDE REGEX /eu/)

file(GLOB_RECURSE CPP_SRCS RELATIVE ${CMAKE_CURRENT_LIST_DIR} src/*.cpp)

set(MOD_FLAGS
    -D_FINALROM=1
    -DF3DEX_GBI_2=1
    -DNAUDIO=1
    -DBUILD_DIR=${BUILD_DIR}
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

set(TOOLS_DIR ${CMAKE_CURRENT_LIST_DIR}/tools)


###############################################################
# preprocess localized text sources for courses and dialogs
###############################################################
# file(GLOB_RECURSE COURSE_INCS RELATIVE ${CMAKE_CURRENT_LIST_DIR}
#     text/*/courses.h
# )
# file(GLOB_RECURSE DIALOGS_INCS RELATIVE ${CMAKE_CURRENT_LIST_DIR}
#     text/*/dialogs.h
# )
set(TEXT_SRCS
    text/define_courses.inc.c
    text/define_text.inc.c
)

# WARNING!!! custom output dir must start with either _,a,b,c,d,e,f,g,h,i,j,k,l,m,n,o
set(text_target_dir "CMakeFiles/_localized_text.dir")
add_custom_target(localized_text)
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
    add_custom_target(${file_out}
        SOURCES ${full_out_path}.preprocessed
        BYPRODUCTS ${full_out_path}
        COMMENT "Generating ${build_out_path}"
        COMMAND
            ${TOOLS_DIR}/textconv ${CMAKE_CURRENT_LIST_DIR}/charmap.txt
            ${full_out_path}.preprocessed
            ${full_out_path}
    )
    add_dependencies(localized_text ${file_out})
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
set(levels_target_dir "CMakeFiles/_levels.dir")
add_custom_target(levels)
set(LEVEL_SRC
    levels/level_headers.h.in
)
cmake_path(GET LEVEL_SRC PARENT_PATH file_dir)
cmake_path(GET LEVEL_SRC STEM LAST_ONLY file_out)
set(out_dir ${LOCALE}/include)

set(out_path ${out_dir}/${file_out}) # e.g. us/include/level_headers.h
set(build_out_path ${levels_target_dir}/${out_path})
set(full_out_path ${CMAKE_CURRENT_BINARY_DIR}/${build_out_path})

add_custom_command(
    OUTPUT ${full_out_path}
    COMMENT "Generating ${build_out_path}"
    COMMAND mkdir -p ${levels_target_dir}/${out_dir}
    COMMAND ${CMAKE_C_PREPROCESSOR} -Wno-trigraphs
    #COMMAND clang -E -P -x c -Wno-trigraphs
    -I${CMAKE_CURRENT_LIST_DIR}/include
    -I${BUILD_DIR}
    -I${BUILD_DIR}/include
    -I${CMAKE_CURRENT_LIST_DIR}/src
    -I${CMAKE_CURRENT_LIST_DIR}
    -I${CMAKE_CURRENT_LIST_DIR}/include/libc
    -DVERSION_US=1 -DF3D_OLD=1
    ${CMAKE_CURRENT_LIST_DIR}/${LEVEL_SRC} | sed -E 's|\(.+\)|\#include \"\\1\"|' > ${full_out_path}
    BYPRODUCTS ${full_out_path}
    MAIN_DEPENDENCY ${CMAKE_CURRENT_LIST_DIR}/${LEVEL_SRC}
)
target_sources(levels PRIVATE ${full_out_path})

##################
# generate images
##################

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
    string(REGEX MATCH "[^.]*$" file_type ${file_type})
    add_custom_command(TARGET PRE-${TARGET_NAME}
        #OUTPUT ${BUILD_DIR}/${file_out}.inc.c
        PRE_BUILD
        COMMENT "Generating ${file_type} src ${file_dir}/${file_out}.inc.c"
        COMMAND mkdir -p ${BUILD_DIR}/${file_dir}
        COMMAND
        ${TOOLS_DIR}/n64graphics -s ${TEXTURE_ENCODING}
        -i ${BUILD_DIR}/${file_dir}/${file_out}.inc.c
        -g ${CMAKE_CURRENT_LIST_DIR}/${IMAGE_SRC}
        -f ${file_type}
        BYPRODUCTS ${BUILD_DIR}/${file_out}.inc.c
        MAIN_DEPENDENCY ${CMAKE_CURRENT_LIST_DIR}/${IMAGE_SRC}
    )
    add_custom_command(TARGET PRE-${TARGET_NAME}
        #OUTPUT ${BUILD_DIR}/${file_out}
        PRE_BUILD
        COMMENT "Generating raw ${file_dir}/${file_out}"
        COMMAND mkdir -p ${BUILD_DIR}/${file_dir}
        COMMAND
        ${TOOLS_DIR}/n64graphics -s raw
        -i ${BUILD_DIR}/${file_dir}/${file_out}
        -g ${CMAKE_CURRENT_LIST_DIR}/${IMAGE_SRC}
        -f ${file_type}
        BYPRODUCTS ${BUILD_DIR}/${file_out}
        MAIN_DEPENDENCY ${CMAKE_CURRENT_LIST_DIR}/${IMAGE_SRC}
    )
endforeach()

# generate image color indexes


# add_library(images OBJECT
#     ${IMAGE_SRCS}
# )
# add_dependencies(${TARGET_NAME} images)


######################
# generate sky images
######################
add_custom_command(TARGET PRE-${TARGET_NAME}
    PRE_BUILD
    COMMENT "Generating ${BUILD_DIR}/levels/ending/cake.inc.c"
    COMMAND ${TOOLS_DIR}/skyconv --type cake --split ${CMAKE_CURRENT_LIST_DIR}/levels/ending/cake.png ${BUILD_DIR}/levels/ending
)
add_custom_command(TARGET PRE-${TARGET_NAME}
    PRE_BUILD
    COMMENT "Generating ${BUILD_DIR}/bin/water_skybox.c"
    COMMAND ${TOOLS_DIR}/skyconv --type sky --split ${CMAKE_CURRENT_LIST_DIR}/textures/skyboxes/water.png ${BUILD_DIR}/bin
)

#####################
# Generate sounds
#####################

############################################
# Generate Level Compressed ELF Load Points
############################################
file(GLOB_RECURSE LEVEL_SRCS RELATIVE ${CMAKE_CURRENT_LIST_DIR}
    levels/*leveldata.c
)
foreach(LEVEL_SRC ${LEVEL_SRCS})
    cmake_path(GET LEVEL_SRC PARENT_PATH level_dir)
    cmake_path(GET LEVEL_SRC STEM level_name)
    string(REGEX REPLACE "/" "." level_target ${LEVEL_SRC})
    message("level_target ${level_target}")

    # TODO: requires textures first
    add_executable(${level_target}
        ${LEVEL_SRC}
    )
    set_target_properties(${level_target} PROPERTIES
        PREFIX ""
        OUTPUT_NAME ${level_name}
        SUFFIX ".elf"
        ARCHIVE_OUTPUT_DIRECTORY ${BUILD_DIR}/${level_dir}
        LIBRARY_OUTPUT_DIRECTORY ${BUILD_DIR}/${level_dir}
        RUNTIME_OUTPUT_DIRECTORY ${BUILD_DIR}/${level_dir}
    )
    target_compile_definitions(${level_target} PRIVATE
        TARGET_N64=1
        _LANGUAGE_C=1
        VERSION_US=1
        DF3D_OLD=1
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
        ${CMAKE_CURRENT_BINARY_DIR}/${BUILD_DIR}
        ${CMAKE_CURRENT_BINARY_DIR}/${BUILD_DIR}/include
        include/libc
        src
        .
    )
    target_link_libraries(${level_target} PRIVATE n64)
    execute_process(
        COMMAND yq -r ".\"texture-bin\"" ${CMAKE_CURRENT_LIST_DIR}/${level_dir}/level.yaml
        OUTPUT_VARIABLE level_texture
        OUTPUT_STRIP_TRAILING_WHITESPACE
    )
    target_link_options(${level_target} PUBLIC
        # Map file currently not used
        -Wl,-e,0,-Ttext=0x09000000,-Map,${level_target}.elf.map,--just-symbols=${level_texture}.elf
    )
    # extract compressible data
    add_custom_command(TARGET ${level_target}
        POST_BUILD
        COMMENT "Extracting to ${level_name}.bin"
        COMMAND ${CMAKE_OBJCOPY} -O binary --only-section=.data
        # TODO: output to BUILD_DIR
        ${BUILD_DIR}/${level_dir}/${level_name}.elf
        ${BUILD_DIR}/${level_dir}/${level_name}.bin
    )
    # compress
    add_custom_command(TARGET ${level_target}
        POST_BUILD
        COMMENT "Compressing to ${level_name}.mio0"
        COMMAND ${TOOLS_DIR}/mio0
        ${BUILD_DIR}/${level_dir}/${level_name}.bin
        ${BUILD_DIR}/${level_dir}/${level_name}.mio0
    )
    # convert MIO00 to ELF.o
    add_custom_command(TARGET ${level_target}
        POST_BUILD
        COMMENT "Converting to ELF ${level_name}.mio0.o"
        COMMAND mips64-elf-ld -r -b binary
        ${BUILD_DIR}/${level_dir}/${level_name}.mio0
        -o ${BUILD_DIR}/${level_dir}/${level_name}.mio0.o
    )
    add_dependencies(PRE-${TARGET_NAME} ${level_target})
endforeach()


############################################
# Levels
# Generate Common Compressed ELF Load Points
############################################
file(GLOB_RECURSE BIN_SRCS RELATIVE ${CMAKE_CURRENT_LIST_DIR}
    bin/*.c
    # actors/group0.c
    # actors/group1.c
    # actors/group2.c
    # actors/group3.c
    # actors/group4.c
    # actors/group5.c
    # actors/group6.c
    # actors/group7.c
    # actors/group8.c
    # actors/group9.c
    # actors/group10.c
    # actors/group11.c
    # actors/group12.c
    # actors/group13.c
    # actors/group14.c
    # actors/group15.c
    # actors/group16.c
    # actors/group17.c
    # actors/common0.c
    # actors/common1.c
)
list(FILTER BIN_SRCS EXCLUDE REGEX .inc.c)
list(FILTER BIN_SRCS EXCLUDE REGEX /eu/)
foreach(BIN_SRC ${BIN_SRCS})
    cmake_path(GET BIN_SRC PARENT_PATH bin_dir)
    cmake_path(GET BIN_SRC STEM bin_name)
    # string(REGEX MATCH "(.*)\\.[^.]+$" bin_target ${BIN_SRC})
    string(REGEX REPLACE "/" "." bin_target ${BIN_SRC})
    message("bin_target ${bin_target}")

    # TODO: requires textures first
    add_executable(${bin_target}
        ${BIN_SRC}
    )
    set_target_properties(${bin_target} PROPERTIES
        PREFIX ""
        OUTPUT_NAME ${bin_name}
        SUFFIX ".elf"
    )
    target_compile_definitions(${bin_target} PRIVATE
        TARGET_N64=1
        _LANGUAGE_C=1
        VERSION_US=1
        DF3D_OLD=1
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
        ${CMAKE_CURRENT_BINARY_DIR}/${BUILD_DIR}
        ${CMAKE_CURRENT_BINARY_DIR}/${BUILD_DIR}/include
        include/libc
        src
        .
    )
    target_link_libraries(${bin_target} PRIVATE n64)
    target_link_options(${bin_target} PUBLIC
        -Wl,-e,0,-Ttext=0x09000000,-Map,${bin_target}.elf.map
    )
    # extract compressible data
    add_custom_command(TARGET ${bin_target}
        POST_BUILD
        COMMENT "Extracting to ${bin_name}.bin"
        COMMAND ${CMAKE_OBJCOPY} -O binary --only-section=.data
        # TODO: output to BUILD_DIR
        ${bin_name}.elf
        ${BUILD_DIR}/${bin_dir}/${bin_name}.bin
    )
    # compress
    add_custom_command(TARGET ${bin_target}
        POST_BUILD
        COMMENT "Compressing to ${bin_name}.mio0"
        COMMAND ${TOOLS_DIR}/mio0
        ${BUILD_DIR}/${bin_dir}/${bin_name}.bin
        ${BUILD_DIR}/${bin_dir}/${bin_name}.mio0
    )
    # convert MIO00 to ELF.o
    add_custom_command(TARGET ${bin_target}
        POST_BUILD
        COMMENT "Converting to ELF ${bin_name}.mio0.o"
        COMMAND mips64-elf-ld -r -b binary
        ${BUILD_DIR}/${bin_dir}/${bin_name}.mio0
        -o ${BUILD_DIR}/${bin_dir}/${bin_name}.mio0.o
    )
    add_dependencies(PRE-${TARGET_NAME} ${bin_target})
endforeach()



###########################
# Generate final project
###########################
add_executable(${TARGET_NAME}
    ${ASM_SRCS}
    ${SOUND_SRCS}
    ${ASSETS_SRCS}
    ${C_SRCS}
    ${CPP_SRCS}
)
add_dependencies(${TARGET_NAME}
    localized_text
    levels
) #PRE-${TARGET_NAME})

set_target_properties(${TARGET_NAME} PROPERTIES
    OUTPUT_NAME "${TARGET_NAME}"
    SUFFIX ".elf"
)
target_link_libraries(${TARGET_NAME} n64)
target_include_directories(${TARGET_NAME} PRIVATE .)
target_include_directories(${TARGET_NAME} PUBLIC
    include
    src
    #sound
    #assets
    ${N64SDK_DIR}/include/n64
    ${N64SDK_DIR}/include/n64/PR
    ${N64SDK_DIR}/include/n64/nusys
    ${CMAKE_SOURCE_DIR}/projects/nucpp/include
    
    # prebuild sources
    ${CMAKE_CURRENT_BINARY_DIR}/${BUILD_DIR} # textures
    ${CMAKE_CURRENT_BINARY_DIR}/${BUILD_DIR}/include # text
)
target_compile_definitions(${TARGET_NAME} PUBLIC 
    #_FINALROM=1
    #F3DEX_GBI_2=1
    #N_AUDIO=1
    MAXWIDTH=320
    MAXHEIGHT=240
    _FORTIFY_SOURCE=2
    NON_MATCHING=1
    AVOID_UB=1
    VERSION_US=1
)

##################
# generate textures
##################
# add_custom_command(TARGET ${TARGET_NAME}
#     PRE_BUILD
#     COMMAND $(TOOLS_DIR)/n64graphics
#     COMMAND $(TOOLS_DIR)/n64graphics_ci
# )
# copy to target/projects/sm64plus

###################
# generate rsp asm
###################
file(GLOB_RECURSE RSP_SRCS RELATIVE ${CMAKE_CURRENT_LIST_DIR} rsp/*.s)
foreach(RSP_SRC ${RSP_SRCS})
    cmake_path(GET RSP_SRC PARENT_PATH file_dir)
    cmake_path(GET RSP_SRC STEM LAST_ONLY file_out)
    add_custom_command(TARGET PRE-${TARGET_NAME}
        PRE_BUILD
        COMMENT "Generating rsp ${file_dir}/${file_out}.bin"
        WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}
        COMMAND mkdir -p ${CMAKE_CURRENT_BINARY_DIR}/${BUILD_DIR}/${file_dir}
        COMMAND
        ${TOOLS_DIR}/armips -sym ${CMAKE_CURRENT_BINARY_DIR}/${BUILD_DIR}/${file_dir}/${file_out}.bin.sym
        -definelabel VERSION_US 1
        -definelabel F3D_OLD 1
        -strequ CODE_FILE ${CMAKE_CURRENT_BINARY_DIR}/${BUILD_DIR}/${file_dir}/${file_out}.bin
        -strequ DATA_FILE ${CMAKE_CURRENT_BINARY_DIR}/${BUILD_DIR}/${file_dir}/${file_out}_data.bin
        ${CMAKE_CURRENT_LIST_DIR}/${RSP_SRC}
        BYPRODUCTS
        ${CMAKE_CURRENT_BINARY_DIR}/${BUILD_DIR}/${file_dir}/${file_out}.bin
        ${CMAKE_CURRENT_BINARY_DIR}/${BUILD_DIR}/${file_dir}/${file_out}.bin.sym
        ${CMAKE_CURRENT_BINARY_DIR}/${BUILD_DIR}/${file_dir}/${file_out}_data.bin
        MAIN_DEPENDENCY ${CMAKE_CURRENT_LIST_DIR}/${RSP_SRC}
    )
endforeach()


###################
# build boot object
###################
add_custom_command(TARGET ${TARGET_NAME}
    PRE_BUILD
    COMMAND ${CMAKE_OBJCOPY} -I binary -B mips -O elf32-bigmips ${BOOT_SRC} ${BUILD_DIR}/${BOOT_OBJ}
    COMMENT "generating boot object ${BUILD_DIR}/${BOOT_OBJ}"
    MAIN_DEPENDENCY ${BOOT_SRC}
    BYPRODUCTS ${BUILD_DIR}/${BOOT_OBJ}
)


#####################
# build linker script with gcc preprocessor
#####################
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

###########
# LINKING
###########
target_link_options(${TARGET_NAME}
    PUBLIC
    ${BUILD_DIR}/${BOOT_OBJ}
    -Wl,-T,${BUILD_DIR}/${LD_SCRIPT},-Map,${BUILD_DIR}/${TARGET_NAME}.map,--no-check-sections,--gc-sections
)
target_link_directories(${TARGET_NAME}
    PUBLIC
    .
    ${BUILD_DIR}
    ${N64SDK_DIR}
)
# order specific
target_link_libraries(${TARGET_NAME}
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
    n64sdkmod::ultra_rom
    gcc
)

######################
# Pad (for bootcode?)
######################
add_custom_command(TARGET ${TARGET_NAME}
    POST_BUILD
    COMMAND ${CMAKE_OBJCOPY} --pad-to=0x100000 --gap-fill=0xFF $<TARGET_FILE:${TARGET_NAME}> ${ROM_NAME} -O binary
    COMMAND makemask ${ROM_NAME}
    COMMAND mips64-elf-size -A -d ${TARGET_NAME}.elf
    COMMAND stat --format='%n: %s bytes' * | grep ${TARGET_NAME}.elf
    BYPRODUCTS ${ROM_NAME}
    COMMENT "Generating ${ROM_NAME}"
)

#################
# Common Failures
#################
# undefined reference to `NU_SPEC_BOOT_STACK`
# - missing "-x assembler-with-cpp" on assembling asm objects

