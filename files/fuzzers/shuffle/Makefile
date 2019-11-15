here := $(realpath $(dir $(lastword $(MAKEFILE_LIST))))

PY_SPEC_HARNESS_PATH := $(here)/pyspec/harness.py
TRINITY_HARNESS_PATH := $(here)/trinity/harness.py

lighthouse_dir_contents := $(shell find $(here)/lighthouse | sed 's/ /\\ /g')

.PHONY: all
all: fuzzer

zrnt_prefix := shuffle_
prysm_prefix := prysm_shuffle_
# NOTE bls disable not necessary here
# worth putting a dummy in for a consistent makefile?


lighthouse.a : lighthouse $(lighthouse_dir_contents) $(CARGO_CONFIG_PATH)
	rm -rf lighthouse.a
	rm -rf lighthouse_out/
	mkdir lighthouse_out/
	# NOTE: we can't pass coverage flags via RUSTFLAGS, so rely on a custom .cargo/config
	# until https://github.com/rust-lang/cargo/issues/6139 is resolved
	cargo build \
		--target-dir=lighthouse_out \
		--manifest-path=lighthouse/Cargo.toml \
		--release
	cp lighthouse_out/release/deps/libswap_or_not_shuffle_fuzzer-*.a lighthouse.a

prysm.a : export GO_PATH := $(COMMON_GOPATH):$(PRYSM_GOPATH)
prysm.a: prysm/fuzz.go
	test $(GO_FUZZ_BUILD_PATH)
	cd prysm && "$(GO_FUZZ_BUILD_PATH)" -libfuzzer-prefix=$(prysm_prefix) -libfuzzer-ex -o ../prysm.a .

zrnt.a : export GO_PATH := $(COMMON_GOPATH):$(ZRNT_GOPATH)
zrnt.a : zrnt/fuzz.go
	test -x $(GO_FUZZ_BUILD_PATH)
	cd zrnt && "$(GO_FUZZ_BUILD_PATH)" -tags preset_mainnet -libfuzzer-prefix=$(zrnt_prefix) -libfuzzer-ex -o ../zrnt.a .

# TODO(gnattishness) make fuzzer rule only link/load, so theres a fuzzer.o rule

# TODO would optimally want this rebuilt if the env var values changed
# https://stackoverflow.com/questions/11647859/make-targets-depend-on-variables
fuzzer : fuzzer.cpp zrnt.a lighthouse.a# TODO move LDFLAGS before and use LDLIBS for all -llibname flags
# TODO how to split up the PYTHON_LD_FLAGS?
fuzzer : fuzzer.cpp zrnt.a lighthouse.a prysm.a
	test $(SHUFFLE_ROOT)
	$(CXX) $(CXXFLAGS) -fsanitize=fuzzer -std=c++17 -DPYTHON_HARNESS_PATH="\"$(PY_SPEC_HARNESS_PATH)\"" -DPYTHON_HARNESS_BIN="\"$(PY_SPEC_BIN_PATH)\"" -DPRYSM_FUZZ_PREFIX=$(prysm_prefix) -DGO_FUZZ_PREFIX=$(zrnt_prefix) fuzzer.cpp zrnt.a lighthouse.a $(LDFLAGS) $(PYTHON_LD_FLAGS) -o fuzzer
	test -d $(TRINITY_VENV_PATH)
	test -d $(PY_SPEC_VENV_PATH)
	$(CXX) $(CXXFLAGS) -fsanitize=fuzzer -std=c++17 \
	    -DPY_SPEC_HARNESS_PATH="\"$(PY_SPEC_HARNESS_PATH)\"" \
	    -DPY_SPEC_VENV_PATH="\"$(PY_SPEC_VENV_PATH)\"" \
	    -DTRINITY_HARNESS_PATH="\"$(TRINITY_HARNESS_PATH)\"" \
	    -DTRINITY_VENV_PATH="\"$(TRINITY_VENV_PATH)\"" \
	    fuzzer.cpp zrnt.a lighthouse.a \
	    $(LDFLAGS) $(LDLIBS) $(PYTHON_LD_FLAGS) -o fuzzer

.PHONY: clean
clean:
	rm -rf fuzzer zrnt.a lighthouse.a lighthouse_out
