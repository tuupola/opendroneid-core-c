add_library(opendroneidcn SHARED opendroneidcn.c)

configure_file(libopendroneidcn.pc.cmake libopendroneidcn.pc @ONLY)

install(TARGETS opendroneidcn DESTINATION lib)
