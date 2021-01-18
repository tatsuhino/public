set(srcs "main.c"
    "board.c" "ble_mesh_init.c")

idf_component_register(SRCS "${srcs}"
                    INCLUDE_DIRS  ".")
