cmake_minimum_required(VERSION 3.25)
set(CMAKE_CXX_STANDARD 14)

project(cryptopp LANGUAGES C CXX ASM)

set(Source_Files
    cryptlib.cpp cpu.cpp integer.cpp 3way.cpp adler32.cpp algebra.cpp
    algparam.cpp allocate.cpp arc4.cpp aria.cpp ariatab.cpp asn.cpp
    authenc.cpp base32.cpp base64.cpp basecode.cpp bfinit.cpp blake2.cpp
    blake2b_simd.cpp blake2s_simd.cpp blowfish.cpp blumshub.cpp camellia.cpp
    cast.cpp casts.cpp cbcmac.cpp ccm.cpp chacha.cpp chacha_avx.cpp
    chacha_simd.cpp chachapoly.cpp cham.cpp cham_simd.cpp channels.cpp
    cmac.cpp crc.cpp crc_simd.cpp darn.cpp default.cpp des.cpp dessp.cpp
    dh.cpp dh2.cpp dll.cpp donna_32.cpp donna_64.cpp donna_sse.cpp dsa.cpp
    eax.cpp ec2n.cpp eccrypto.cpp ecp.cpp elgamal.cpp emsa2.cpp eprecomp.cpp
    esign.cpp files.cpp filters.cpp fips140.cpp fipstest.cpp gcm.cpp
    gcm_simd.cpp gf256.cpp gf2_32.cpp gf2n.cpp gf2n_simd.cpp gfpcrypt.cpp
    gost.cpp gzip.cpp hc128.cpp hc256.cpp hex.cpp hight.cpp hmac.cpp
    hrtimer.cpp ida.cpp idea.cpp iterhash.cpp kalyna.cpp kalynatab.cpp
    keccak.cpp keccak_core.cpp keccak_simd.cpp lea.cpp lea_simd.cpp
    lsh256.cpp lsh256_avx.cpp lsh256_sse.cpp lsh512.cpp lsh512_avx.cpp
    lsh512_sse.cpp luc.cpp mars.cpp marss.cpp md2.cpp md4.cpp md5.cpp
    misc.cpp modes.cpp mqueue.cpp mqv.cpp nbtheory.cpp oaep.cpp osrng.cpp
    padlkrng.cpp panama.cpp pkcspad.cpp poly1305.cpp polynomi.cpp
    ppc_simd.cpp primetab.cpp pssr.cpp pubkey.cpp queue.cpp rabbit.cpp
    rabin.cpp randpool.cpp rc2.cpp rc5.cpp rc6.cpp rdtables.cpp
    rijndael.cpp rijndael_simd.cpp ripemd.cpp rng.cpp rsa.cpp rw.cpp
    safer.cpp salsa.cpp scrypt.cpp seal.cpp seed.cpp serpent.cpp sha.cpp
    sha3.cpp sha_simd.cpp shacal2.cpp shacal2_simd.cpp shake.cpp shark.cpp
    sharkbox.cpp simeck.cpp simon.cpp simon128_simd.cpp skipjack.cpp sm3.cpp
    sm4.cpp sm4_simd.cpp sosemanuk.cpp speck.cpp speck128_simd.cpp
    square.cpp squaretb.cpp sse_simd.cpp strciphr.cpp tea.cpp tftables.cpp
    threefish.cpp tiger.cpp tigertab.cpp ttmac.cpp tweetnacl.cpp twofish.cpp
    vmac.cpp wake.cpp whrlpool.cpp xed25519.cpp xtr.cpp xtrcrypt.cpp xts.cpp
    zdeflate.cpp zinflate.cpp zlib.cpp)

#@formatter:off
set(BUILDING_DEBUG   (${CMAKE_BUILD_TYPE} MATCHES "Debug"))
set(BUILDING_RELEASE (NOT ${BUILDING_DEBUG}))
set(ARCH_ARM         (${CMAKE_SYSTEM_PROCESSOR} MATCHES "arm"))
set(ARCH_ARM64       (${CMAKE_SYSTEM_PROCESSOR} MATCHES "arm64"))
set(ARCH_AARCH64     (${CMAKE_SYSTEM_PROCESSOR} MATCHES "aarch64"))
set(BUILDING_ARM     (${ARCH_ARM} OR ${ARCH_AARCH64} OR ${ARCH_ARM64}))
set(BUILDING_64BIT   (${CMAKE_SIZEOF_VOID_P} EQUAL 8))
set(BUILDING_32BIT   (NOT ${BUILDING_64BIT}))
#@formatter:on

if (${BUILDING_ARM})
    set(Source_Files ${Source_Files} neon_simd.cpp)
else ()
    set(Source_Files ${Source_Files} rdrand.cpp)
endif ()

add_library(cryptlib STATIC ${Source_Files})

macro(compile_asm file options)
    if (${BUILDING_64BIT})
        set(ASM_COMPILER ml64)
    else ()
        set(ASM_COMPILER ml)
    endif ()
    set(asm_file "${CMAKE_CURRENT_LIST_DIR}/${file}")
    set(obj_file "${CMAKE_CURRENT_BINARY_DIR}/${file}.obj")

    add_custom_command(
        OUTPUT ${obj_file}
        COMMAND ${ASM_COMPILER} ${options} /Fo ${obj_file} /c ${asm_file}
        DEPENDS ${asm_file}
        COMMENT "Assembling ${file}")

    target_sources(cryptlib PRIVATE ${obj_file})
endmacro()

if (WIN32)
    if (${BUILDING_64BIT})
        compile_asm(x64masm.asm /nologo /D_M_X64 /W3 /Cx /Zi)
        compile_asm(x64dll.asm /nologo /D_M_X64 /W3 /Zi)
        compile_asm(cpuid64.asm /nologo /D_M_X64 /W3 /Zi)
    else ()
        target_compile_definitions(cryptlib PRIVATE CRYPTOPP_DISABLE_ASM)
    endif ()
endif ()

target_compile_options(cryptlib PRIVATE -maes)

if (${BUILDING_DEBUG})
    target_compile_definitions(cryptlib PRIVATE DEBUG _DEBUG)
    target_link_options(cryptlib PRIVATE /DEBUG)
else ()
    target_compile_definitions(cryptlib PRIVATE NDEBUG _NDEBUG)
    target_link_options(cryptlib PRIVATE /OPT:REF)
endif ()

if (WIN32)
    target_compile_options(cryptlib PRIVATE /FI sdkddkver.h -maes)

    if (${BUILDING_DEBUG})
        target_compile_options(cryptlib PRIVATE /Oi /Oy- /Od /MTd)
    else ()
        target_compile_options(cryptlib PRIVATE /Oi /Oy /O2 /MT)
    endif ()

    if (${BUILDING_32BIT})
        target_compile_options(cryptlib PRIVATE /arch:SSE2)
    else ()
        target_compile_options(cryptlib PRIVATE /arch:AVX2)
    endif ()

    target_link_libraries(cryptlib PRIVATE bcrypt)
endif ()
