# sPHENIX-OO
This repo details all of the work for OO substructure as well as some UE in both simulation and data. The code relies on the sPHENIX repo frameworks, so in order to use them, you'll need the associated libraries that come with those repos from sPHENIX. If possible, log in to SCDF with your BNL credentials.

## To enter SCDF:
`ssh -X <username>@ssh.sdcc.bnl.gov`
## ssh into an available sPHENIX node. [02-08] are possible nodes. It will prompt you to enter your BNL user-associated password:
` ssh sphnxuser05`
## Enter pwd, you will be in your /u/ directory. This is a good place to `git clone` the sPHENIX repos to build your libraries.
## If you move to the /user/ directory instead of /u/, you'll have more space to run and make your own codes. e.g.,
`/sphenix/u/<username>`
`/sphenix/user/<username>`
## If you go to `/sphenix/`directory, you'll find more folders that store simulation files, data files, as well as places that have extra room for holding large root files. e.g., the jet group uses:
`/sphenix/tg/tg01/jets`
## Go ahead and make your own personal directory in here. Now you have a personal place to store files.
`cd /sphenix/tg/tg01/jets`
`mkdir <usernmae>`


# Getting Started with jet substructure:

## O+O jet substructure, run 35 simulation (reference sPHENIX wiki)

Pythia8 jets embedded in HIJING O+O minimum bias. Reco = subtracted calorimeter towers.
Observables: R_g, z_g of anti-kT R=0.4 jets reclustered from R_sub = 0.15 or 0.2 subjets.
Branch list explanations: `OO_substructure_v2_tree.xlsx`.

## Paths

| What | Path |
|---|---|
| Release / install | `ana.576`, `MYINSTALL=/sphenix/user/jamesj3j3/sPHENIX/install` |
| Production module | `/sphenix/user/jamesj3j3/OOInspection/src/OOSubstructureV2.{h,cc}` |
| Production macro, Condor files | `MACRODIR` = `/sphenix/user/jamesj3j3/OOsub/` (`Fun4All_OOSubstructure_emb_v2.C`, `run_ssv2.sh`, `run_ssv2.sub`, `jobs_grouped.txt`, `checkV2.C`, `checkProd.sh`, `checkUE.C`, `mergeProd.sh`) |
| Per-job output | `/sphenix/tg/tg01/jets/jamesj3j3/OOssv2Output/sim_run35_oo_<label>_<seg>_ssv2.root` |
| Merged output | `/sphenix/tg/tg01/jets/jamesj3j3/OOssv2Output/merged/sim_run35_oo_<label>_ss.root` |
| Offline analysis | `ANADIR` = `/sphenix/tg/tg01/jets/jamesj3j3/OOssv2Output/merged` (`ssv2ana.cc`, `JetMatch.h`, `plotSSV2.C`, `runSSV2.sh`) |

`<label>` = `mb`, `jet5`, `jet12`, `jet20`, `jet30`, `jet40`, `jet50`.

## 0. Environment (every new shell) -- this will help if you don't have your own sPHENIX libraries installed

```
source /opt/sphenix/core/bin/sphenix_setup.sh -n ana.576
export MYINSTALL=/sphenix/user/jamesj3j3/sPHENIX/install
source /opt/sphenix/core/bin/setup_local.sh $MYINSTALL
```

## 1. Build the production module (only after changing OOSubstructureV2)

Standard sPHENIX autotools build of OOInspection into `$MYINSTALL`. `Makefile.am` must list
`OOSubstructureV2.h/.cc` and link `-lfastjettools`.

```
cd /sphenix/user/jamesj3j3/OOInspection/src/build
echo $MYINSTALL #check if your install directory exists
../autogen --prefix=$MYINSTALL
make -j8 install 
```

## 2. Production (Condor)

One job = one row of `jobs_grouped.txt`: `pthat seg calo truth global mbdepd`.

Test one job first:
```
cd $MACRODIR
./run_ssv2.sh 20 test <calo> <truth> <global> <mbdepd> ./test 200
root -l -b -q 'checkV2.C("test/sim_run35_oo_jet20_test_ssv2.root")'
```

`run_ssv2.sh pthat seg calo truth global mbdepd outdir [nEvents=0] [windowOn=1] [windowMargin]`

| Argument | Meaning |
|---|---|
| `pthat` | 0 (MB), 5, 12, 20, 30, 40, 50 |
| `seg` | segment tag, used in the output name |
| `calo truth global mbdepd` | input DST lists |
| `outdir` | output directory |
| `nEvents` | 0 = all |
| `windowOn` | 1 = apply the truth lead-jet window |
| `windowMargin` | GeV added on each side; default 3 for pthat <= 20, 0 above |

Submit everything:
```
mkdir -p logs_v2
condor_submit run_ssv2.sub
```

## 3. Check and merge

```
./checkProd.sh jobs_grouped.txt /sphenix/tg/tg01/jets/jamesj3j3/OOssv2Output ssv2
./mergeProd.sh /sphenix/tg/tg01/jets/jamesj3j3/OOssv2Output ssv2 [nproc=8]
```
- `checkProd.sh` writes `jobs_redo.txt` (rows to resubmit) and prints N_processed / N_written per sample.
- `mergeProd.sh` refuses to run while `jobs_redo.txt` is non-empty. Output: `OOssv2Output/merged/sim_run35_oo_<label>_ssv2.root`.
- The current merged set is named `_ss` (MB `_ssv2`, linked to `_ss`).

## 4. Offline analysis

```
cd $ANADIR
./runSSV2.sh <pattern> [nAna=50000] [subMin=1.0] [zcut=0.1] [tag] [recoJES=0.72] [ref=S] [evtLead=10] [minSub=2]
```

| Argument | Meaning |
|---|---|
| `pattern` | merged-file path with `%s` for the label |
| `nAna` | analysis events per sample; 0 = all |
| `subMin` | subjet threshold, true pT [GeV]; reco cut = subMin x recoJES |
| `zcut` | soft-drop z_cut (beta = 0) |
| `tag` | names the outputs; default `sub<subMin>_z<zcut>_ref<ref>_ev<evtLead>_ns<minSub>` |
| `recoJES` | reco / true subjet pT used for the reco cut |
| `ref` | truth reference: S (rhoA-subtracted Pythia+HIJING, nominal), P, U, C |
| `evtLead` | event cut on the leading Pythia R=0.4 jet [GeV]; jets in plots always >= 15 GeV |
| `minSub` | minimum subjets per reclustered jet (2; 1 keeps single-subjet jets) |

Quick test, then full statistics:
```
P=/sphenix/tg/tg01/jets/jamesj3j3/OOssv2Output/merged/sim_run35_oo_%s_ss.root
rm -f ssv2ana
./runSSV2.sh $P 2000 1.0 0.1 quick 0.72 S 10
nohup ./runSSV2.sh $P 0 1.0 0.1 full_ev10 0.72 S 10 > run_full_ev10.log 2>&1 &
```
`runSSV2.sh` builds `ssv2ana` when the source is newer, runs all samples in parallel, merges, and plots.
Progress: `tail -n1 logs_<tag>/*.log`.

Outputs:

| File | Content |
|---|---|
| `logs_<tag>/<label>.log` | per-sample log; last line = analysis events |
| `out_<tag>/ssv2ana_<label>.root` | per-sample histograms |
| `out_<tag>/ssv2ana_all.root` | merged histograms + unfolding tree `pairs` |
| `ssv2ana_<tag>.pdf` | all plot pages |

By hand, one sample, and replotting:
```
./ssv2ana <merged file> <label> <out.root> [nAna] [subMin] [zcut] [recoJES] [ref] [evtLead] [minSub]
root -l -b -q 'plotSSV2.C("out_<tag>/ssv2ana_all.root","name")'
```

## Notes

- Combine jet samples with weight `xsecPb / hEvtCount[bin 1]`, and always require `inWindow`.
- Reco is uncorrected (no JES/JER). True-pT thresholds convert to reco by multiplying by 0.72 (baseline for now).
- Use the stored towers and truth particles for the analysis. The production subjets (`rc_`, `rc15_`) are cut at 1 GeV reco and are a cross-check only.
