add_library(opendroneidgb SHARED opendroneidgb.c)

configure_file(libopendroneidgb.pc.cmake libopendroneidgb.pc @ONLY)

install(TARGETS opendroneidgb DESTINATION lib)
