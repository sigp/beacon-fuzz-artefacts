# "panic: err blsPublicKeyDeserialize" in ZRNT - util/bls/bls_util_on.go


## root cause

https://github.com/protolambda/zrnt/blob/master/eth2/util/bls/bls_util_on.go#L34

## Comparaison with lighthouse

In lighthouse, both ssz are decoded properly.
Then `process_attester_slashing` succeed without any error. 

Maybe we are not checking bls publicKey in lighhouse??

## backtrace

``` sh
./debug_attester_slashing crashes/panic_zrnt_blsPublicKeyDeserialize_beaconstate.ssz crashes/panic_zrnt_blsPublicKeyDeserialize_attester_slashing.ssz 
BeaconState ReadFile OK:  crashes/panic_zrnt_blsPublicKeyDeserialize_beaconstate.ssz
BeaconState decoding OK
AttesterSlashing ReadFile OK
AttesterSlashing ReadFile OK:  crashes/panic_zrnt_blsPublicKeyDeserialize_attester_slashing.ssz
AttesterSlashing decoding OK
InputAttesterSlashing OK
NewFullFeaturedState OK
LoadPrecomputedData OK
panic: err blsPublicKeyDeserialize 000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000

goroutine 1 [running]:
github.com/protolambda/zrnt/eth2/util/bls.parsePubkeys(0xc000142000, 0x8, 0x8, 0x77da5cebfca269f, 0xc8758c6ebdd8c569, 0x39cd53e2ae4b3214)
	/home/scop/Documents/consulting/sigmaprime/fuzzing_sigp/go/src/github.com/protolambda/zrnt/eth2/util/bls/bls_util_on.go:34 +0xe4
github.com/protolambda/zrnt/eth2/util/bls.FastAggregateVerify(0xc000142000, 0x8, 0x8, 0xc8758c6ebdd8c569, 0x39cd53e2ae4b3214, 0x4f9c06f974c494d0, 0x77da5cebfca269f, 0x6f78566ace974391, 0x238056d64078ffda, 0xf4feb56063d4d61d, ...)
	/home/scop/Documents/consulting/sigmaprime/fuzzing_sigp/go/src/github.com/protolambda/zrnt/eth2/util/bls/bls_util_on.go:41 +0xb2
github.com/protolambda/zrnt/eth2/beacon/attestations.(*IndexedAttestation).Validate(0xc000e9a1e8, 0x7fef8025d448, 0xc000078000, 0x7fef8025d448, 0xc000078000)
	/home/scop/Documents/consulting/sigmaprime/fuzzing_sigp/go/src/github.com/protolambda/zrnt/eth2/beacon/attestations/indexed.go:70 +0x4ee
github.com/protolambda/zrnt/eth2/beacon/slashings/attslash.(*AttestSlashFeature).ProcessAttesterSlashing(0xc000078120, 0xc000e9a1e8, 0xc0006dfab0, 0x1)
	/home/scop/Documents/consulting/sigmaprime/fuzzing_sigp/go/src/github.com/protolambda/zrnt/eth2/beacon/slashings/attslash/attester_slashing.go:53 +0xa3
main.main()
	/home/scop/Documents/consulting/sigmaprime/fuzzing_sigp/go/debug_attester_slashing.go:80 +0x8de
```

## debug code

``` golang
// +build preset_mainnet 

package main
	
import (
    //"bufio"
    "fmt"
    "bytes"
    //"io"
    "io/ioutil"
    "os"
)

import (
	"github.com/protolambda/zrnt/eth2/phase0"
	"github.com/protolambda/zrnt/eth2/beacon/slashings/attslash"
	"github.com/protolambda/zssz"
	//"github.com/protolambda/zssz/types"
	//"helper"
)

func check(e error) {
    if e != nil {
        fmt.Println("Error:", e)
        //panic(e)
    }
}

// Input passed to implementations after preprocessing
type InputAttesterSlashing struct {
	Pre              phase0.BeaconState
	AttesterSlashing attslash.AttesterSlashing
}

func main() {

	var state phase0.BeaconState
	var attest attslash.AttesterSlashing

	file_name := os.Args[1]

    data, err := ioutil.ReadFile(file_name)
    check(err)
    fmt.Println("BeaconState ReadFile OK: ", file_name)

    reader := bytes.NewReader(data)

	if err := zssz.Decode(reader, uint64(len(data)), &state, phase0.BeaconStateSSZ); err != nil {
		fmt.Println("Cannot decode prestate %v: %v", file_name, err)
		return
	}

    fmt.Println("BeaconState decoding OK")

	file_name2 := os.Args[2]

	data2, err := ioutil.ReadFile(file_name2)
    check(err)
    fmt.Println("AttesterSlashing ReadFile OK")
    fmt.Println("AttesterSlashing ReadFile OK: ", file_name2)

    reader2 := bytes.NewReader(data2)

	if err := zssz.Decode(reader2, uint64(len(data2)), &attest, attslash.AttesterSlashingSSZ); err != nil {
		fmt.Println("Decoding that should always succeed failed: %v", err)
		return
	}

    fmt.Println("AttesterSlashing decoding OK")

    // TODO - remove usage of this structure 
	var input = InputAttesterSlashing {Pre: state, AttesterSlashing: attest}
	fmt.Println("InputAttesterSlashing OK")

	ffstate := phase0.NewFullFeaturedState(&input.Pre)
	fmt.Println("NewFullFeaturedState OK")
	ffstate.LoadPrecomputedData()
	fmt.Println("LoadPrecomputedData OK")

	error := ffstate.ProcessAttesterSlashing(&input.AttesterSlashing)
	check(error)

	fmt.Println("ProcessAttestation OK")
}

// compilation
// go build -tags preset_mainnet debug_attester_slashing.go
```
