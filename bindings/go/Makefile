ifeq ($(OS),Windows_NT)
UNAME_S ?= Windows_NT
UNAME_P ?= $(PROCESSOR_ARCHITECTURE)
UNAME_M ?= $(PROCESSOR_ARCHITECTURE)
PATH_SEP := ;
EXE := .exe
MKDIR_P := cmake -E make_directory
RM_RF := cmake -E rm -rf
OPENMP_DLL := $(strip $(shell where.exe libgomp-1.dll 2> NUL))
RUN_PATH := $(abspath ../../${BUILD_DIR})$(PATH_SEP)$(abspath ../../${BUILD_DIR}/ggml/src)
CUDA_LIB_DIR :=
CUDA_BIN_DIR :=
CUDA_PATH_RAW := $(strip $(CUDA_PATH))
CUDA_LIB_DIR_OK :=
CUDA_BIN_DIR_OK :=
ifneq ($(CUDA_PATH_RAW),)
CUDA_PATH_WIN := $(subst \\,/,$(CUDA_PATH_RAW))
CUDA_LIB_DIR := $(CUDA_PATH_WIN)/lib/x64
CUDA_BIN_DIR := $(CUDA_PATH_WIN)/bin
CUDA_LIB_DIR_OK := $(strip $(shell if test -d "$(CUDA_LIB_DIR)"; then echo yes; fi))
CUDA_BIN_DIR_OK := $(strip $(shell if test -d "$(CUDA_BIN_DIR)"; then echo yes; fi))
endif
else
ifndef UNAME_S
UNAME_S := $(shell uname -s)
endif

ifndef UNAME_P
UNAME_P := $(shell uname -p)
endif

ifndef UNAME_M
UNAME_M := $(shell uname -m)
endif

PATH_SEP := :
EXE :=
MKDIR_P := install -d
RM_RF := rm -fr
endif

GGML_METAL_PATH_RESOURCES := $(abspath ../..)
BUILD_DIR := build_go
MODELS_DIR := models
EXAMPLES_DIR := $(wildcard examples/*)
INCLUDE_PATH := $(abspath ../../include)$(PATH_SEP)$(abspath ../../ggml/include)
LIBRARY_PATH := $(abspath ../../${BUILD_DIR}/src)$(PATH_SEP)$(abspath ../../${BUILD_DIR}/ggml/src)

ifeq ($(GGML_CUDA),1)
ifeq ($(OS),Windows_NT)
ifeq ($(CUDA_PATH_RAW),)
$(error Env CUDA_PATH Not found, Please set /path/to/ur/CUDA Toolkit)
endif
ifneq ($(CUDA_LIB_DIR_OK),)
	LIBRARY_PATH := $(LIBRARY_PATH)$(PATH_SEP)$(CUDA_LIB_DIR)
else
$(error Dir $(CUDA_LIB_DIR) not found, make sure CUDA is properly installed with correct CUDA_PATH)
endif
ifneq ($(CUDA_BIN_DIR_OK),)
RUN_PATH := $(RUN_PATH)$(PATH_SEP)$(CUDA_BIN_DIR)
endif
else
	LIBRARY_PATH := $(LIBRARY_PATH)$(PATH_SEP)$(CUDA_PATH)/targets/$(UNAME_M)-linux/lib/
endif
	BUILD_FLAGS := -ldflags "-extldflags '-lcudart -lcuda -lcublas'"
endif

ifeq ($(UNAME_S),Darwin)
	LIBRARY_PATH := $(LIBRARY_PATH)$(PATH_SEP)$(abspath ../../${BUILD_DIR}/ggml/src/ggml-blas)$(PATH_SEP)$(abspath ../../${BUILD_DIR}/ggml/src/ggml-metal)
	EXT_LDFLAGS := -framework Foundation -framework Metal -framework MetalKit -lggml-metal -lggml-blas
endif

all: clean whisper examples

whisper: mkdir
	cmake -S ../.. -B ../../${BUILD_DIR} \
		-DCMAKE_BUILD_TYPE=Release \
		-DBUILD_SHARED_LIBS=OFF
	cmake --build ../../${BUILD_DIR} --target whisper
ifeq ($(OS),Windows_NT)
	@cmake -E copy_if_different ../../${BUILD_DIR}/ggml/src/ggml.a ../../${BUILD_DIR}/ggml/src/libggml.a
	@cmake -E copy_if_different ../../${BUILD_DIR}/ggml/src/ggml-base.a ../../${BUILD_DIR}/ggml/src/libggml-base.a
	@cmake -E copy_if_different ../../${BUILD_DIR}/ggml/src/ggml-cpu.a ../../${BUILD_DIR}/ggml/src/libggml-cpu.a
ifneq ($(OPENMP_DLL),)
	@cmake -E copy_if_different "$(OPENMP_DLL)" ../../${BUILD_DIR}/
endif
endif

test: model-small whisper modtidy
ifeq ($(UNAME_S),Darwin)
	@C_INCLUDE_PATH="$(INCLUDE_PATH)" LIBRARY_PATH="$(LIBRARY_PATH)" GGML_METAL_PATH_RESOURCES="$(GGML_METAL_PATH_RESOURCES)" go test -ldflags "-extldflags '$(EXT_LDFLAGS)'" -v .
	@C_INCLUDE_PATH="$(INCLUDE_PATH)" LIBRARY_PATH="$(LIBRARY_PATH)" GGML_METAL_PATH_RESOURCES="$(GGML_METAL_PATH_RESOURCES)" go test -ldflags "-extldflags '$(EXT_LDFLAGS)'" -v ./pkg/whisper/...
else
ifeq ($(OS),Windows_NT)
	@PATH="$(RUN_PATH)$(PATH_SEP)$$PATH" C_INCLUDE_PATH="$(INCLUDE_PATH)" LIBRARY_PATH="$(LIBRARY_PATH)" go test -v .
	@PATH="$(RUN_PATH)$(PATH_SEP)$$PATH" C_INCLUDE_PATH="$(INCLUDE_PATH)" LIBRARY_PATH="$(LIBRARY_PATH)" go test -v ./pkg/whisper/...
else
	@C_INCLUDE_PATH="$(INCLUDE_PATH)" LIBRARY_PATH="$(LIBRARY_PATH)" go test -v .
	@C_INCLUDE_PATH="$(INCLUDE_PATH)" LIBRARY_PATH="$(LIBRARY_PATH)" go test -v ./pkg/whisper/...
endif
endif

examples: $(EXAMPLES_DIR)

model-small: mkdir examples/go-model-download
	@${BUILD_DIR}/go-model-download$(EXE) -out models ggml-small.en.bin

$(EXAMPLES_DIR): mkdir whisper modtidy
ifeq ($(UNAME_S),Darwin)
	@C_INCLUDE_PATH="$(INCLUDE_PATH)" LIBRARY_PATH="$(LIBRARY_PATH)" GGML_METAL_PATH_RESOURCES="$(GGML_METAL_PATH_RESOURCES)" go build ${BUILD_FLAGS} -ldflags "-extldflags '$(EXT_LDFLAGS)'" -o ${BUILD_DIR}/$(notdir $@)$(EXE) ./$@
else
ifeq ($(OS),Windows_NT)
	@PATH="$(RUN_PATH)$(PATH_SEP)$$PATH" C_INCLUDE_PATH="$(INCLUDE_PATH)" LIBRARY_PATH="$(LIBRARY_PATH)" go build ${BUILD_FLAGS} -o ${BUILD_DIR}/$(notdir $@)$(EXE) ./$@
else
	@C_INCLUDE_PATH="$(INCLUDE_PATH)" LIBRARY_PATH="$(LIBRARY_PATH)" go build ${BUILD_FLAGS} -o ${BUILD_DIR}/$(notdir $@)$(EXE) ./$@
endif
endif

mkdir:
	@${MKDIR_P} ${BUILD_DIR}
	@${MKDIR_P} ${MODELS_DIR}

modtidy:
	@go mod tidy

clean:
	@${RM_RF} $(BUILD_DIR)
	@go clean
