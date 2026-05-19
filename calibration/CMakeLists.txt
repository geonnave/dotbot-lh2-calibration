# calibration — LH2 calibration firmware streaming HDLC frames over UART.
#
# Consumes the dotbot_libs INTERFACE target for startup/BSP/board, plus
# adds the lh2/uart/hdlc sources that aren't (yet) in dotbot_libs's
# INTERFACE target_sources. When dotbot-libs's INTERFACE grows to
# include those, the explicit list below can shrink.

set(LINKER_SCRIPT ${CMAKE_SOURCE_DIR}/cmake/targets/nrf5340-app/nrf5340_app.ld)

add_executable(calibration
    main.c
    # Extra dotbot-libs sources beyond the dotbot_libs INTERFACE target.
    # lh2.c is a dispatcher that #include's lh2_default.c for app-core builds.
    ${DOTBOT_LIBS}/bsp/nrf/lh2.c
    ${DOTBOT_LIBS}/bsp/nrf/lh2_decoder.c
    ${DOTBOT_LIBS}/bsp/nrf/uart.c
    ${DOTBOT_LIBS}/bsp/nrf/timer_hf.c   # lh2_default uses db_timer_hf_*
    ${DOTBOT_LIBS}/drv/hdlc/hdlc.c
)

target_link_libraries(calibration PRIVATE dotbot_libs)

target_include_directories(calibration PRIVATE
    ${NRFX_MDK_INCLUDE}
    ${CMSIS_CORE_INCLUDE}
    ${DOTBOT_LIBS}/drv/hdlc          # for hdlc.h (its own folder, not flat under drv/)
)

target_compile_definitions(calibration PRIVATE
    NRF5340_XXAA
    NRF_APPLICATION
    __NRF_FAMILY
    BOARD_DOTBOT_V3
)

target_link_options(calibration PRIVATE
    -T${LINKER_SCRIPT}
    -Wl,-Map=${CMAKE_CURRENT_BINARY_DIR}/calibration.map
)

set_target_properties(calibration PROPERTIES LINK_DEPENDS ${LINKER_SCRIPT})

add_custom_command(TARGET calibration POST_BUILD
    COMMAND ${CMAKE_OBJCOPY} -O ihex   $<TARGET_FILE:calibration> ${CMAKE_CURRENT_BINARY_DIR}/calibration.hex
    COMMAND ${CMAKE_OBJCOPY} -O binary $<TARGET_FILE:calibration> ${CMAKE_CURRENT_BINARY_DIR}/calibration.bin
    COMMAND ${CMAKE_SIZE} $<TARGET_FILE:calibration>
    BYPRODUCTS ${CMAKE_CURRENT_BINARY_DIR}/calibration.hex
               ${CMAKE_CURRENT_BINARY_DIR}/calibration.bin
    COMMENT "calibration: hex + bin"
)
