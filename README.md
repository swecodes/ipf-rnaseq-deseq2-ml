# Transcriptomic Signatures of Idiopathic Pulmonary Fibrosis: Statistical Testing, DESeq2, and Machine Learning

A bulk RNA-seq analysis comparing lung tissue from patients with idiopathic pulmonary fibrosis (IPF) against healthy controls. The project builds the analysis from first principles (normalization, per-gene hypothesis testing, multiple-testing correction), then checks those results against DESeq2, the standard count-based model for RNA-seq, and finally asks a machine learning question: can gene expression alone identify IPF, and which genes carry that signal?

## Background

IPF is a progressive, irreversible scarring of the lung with no known cause and limited treatment options. Because the disease is driven by changes in how lung cells behave, comparing gene expression between diseased and healthy tissue is a direct way to see which biological programs are switched on or off. The goal here is not only to produce a list of genes, but to show that the list is statistically sound, consistent across methods, and biologically sensible.

## Data

GEO accession [GSE150910](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE150910) (Furusawa et al., 2020): whole-lung RNA-seq from patients with chronic hypersensitivity pneumonitis (CHP), IPF, and controls. This analysis uses the 103 IPF and 103 control samples; CHP samples are excluded. The gene-level count file contains 18,838 genes, of which 3,183 pass the low-expression filter.

## Approach

**1. Normalization and filtering.** Raw counts are scaled for sequencing depth and genes with low mean expression in both groups are removed, since they carry little information and add to the multiple-testing burden.

**2. Per-gene testing.** Each gene is first tested with the Kolmogorov–Smirnov test, which detects any difference between the two distributions. Because the question of interest is a shift in average expression, Welch's t-test and the Mann–Whitney U test are also run on log2(CPM + 1). Effect size is reported as log2 fold change.

**3. Multiple-testing correction.** With thousands of genes tested at once, a 0.05 threshold would produce many false positives by chance. Both Bonferroni (controls the chance of any false hit) and Benjamini–Hochberg (controls the expected fraction of false hits) are compared.

**4. DESeq2.** RNA-seq counts are discrete and their variance grows with the mean, which simple tests ignore. PyDESeq2 models counts with a negative binomial distribution, estimates its own size factors, and shrinks noisy dispersion estimates across genes. It serves as the reference method against which the simpler tests are compared.

**5. Quality control.** PCA on log-expression shows whether disease status is a dominant source of variation and flags samples that sit far from their group. Library sizes are compared between groups to check for depth bias.

**6. Classification.** L1-regularized logistic regression and a random forest are trained to separate IPF from control using 5-fold stratified cross-validation. Gene selection is performed *inside* each training fold, so held-out samples never influence which genes are chosen. Selecting genes on the full dataset first is a common form of data leakage that inflates reported performance.

**7. Biological interpretation.** Results are checked against genes already associated with IPF, and the up- and down-regulated gene sets are tested for pathway enrichment (GO Biological Process, KEGG) using Enrichr.

## Results

**Differential expression.** At a BH-adjusted FDR of 0.05, the three simple tests agree closely (KS: 1,901 genes; Welch: 1,895; Mann–Whitney: 1,915), while Bonferroni retains 933. DESeq2 finds 1,960 genes at FDR < 0.05, of which 205 also change at least two-fold (121 up, 84 down in IPF). The log2 fold changes from the simple approach and from DESeq2 are almost perfectly rank-correlated (Spearman ρ = 0.999), and 1,723 genes are significant under both KS and DESeq2, indicating the findings are not an artifact of one method.

**Known biology is recovered.** Established IPF-associated genes are strongly up-regulated: MUC5B (log2FC 3.52), SPP1 (2.32), COL1A1 (2.32), COL3A1 (2.23), and CXCL14 (2.07), all with adjusted p-values below 1e-14.

**Sample structure.** IPF and control samples separate along PC1 (23% of variance) with partial overlap, consistent with a real but heterogeneous disease signal. Four samples were flagged as possible outliers (three control, one IPF) and retained.

**Classification.** Both models separate IPF from control well under leakage-free cross-validation:

| Model | ROC-AUC | Accuracy | Balanced accuracy | F1 |
|---|---|---|---|---|
| L1 logistic regression | 0.983 ± 0.030 | 0.946 ± 0.050 | 0.946 ± 0.050 | 0.947 ± 0.047 |
| Random forest | 0.983 ± 0.027 | 0.932 ± 0.052 | 0.931 ± 0.053 | 0.932 ± 0.050 |

The L1 model relies on a compact 21-gene panel, 13 of which are also DESeq2 hits. Its strongest IPF-associated weight is on POSTN (periostin), a known marker of fibrotic activity.

**Pathways.** Up-regulated genes are enriched for extracellular matrix and collagen fibril organization, ECM–receptor interaction, and focal adhesion, which is the expected signature of tissue scarring. Down-regulated genes are enriched for angiogenesis and TGF-β response terms. The latter is notable because TGF-β is a central pro-fibrotic signal; this likely reflects loss of specific cell populations (for example, vascular cells) rather than reduced TGF-β activity, and is an open question for follow-up.

## Next steps

- **CHP vs IPF.** The clinically harder and more meaningful problem, and the question this dataset was originally designed to address. The two diseases can look similar clinically, and a reliable molecular distinction would be useful.
- Add clinical covariates to the DESeq2 design (`~ sex + age + smoking + condition`).
- Validate the gene signature and classifier on an independent IPF cohort.

## Reproducing

Download the gene-level count file from GEO and update the file path in the notebook. Install dependencies, then restart the runtime before running (PyDESeq2 requires a newer pandas than Colab loads by default):

    pip install numpy scipy "pandas>=2.3" scikit-learn matplotlib seaborn pydeseq2 gseapy

The enrichment step calls the Enrichr web service and requires internet access.

## Reference

Furusawa H, et al. Chronic Hypersensitivity Pneumonitis, an Interstitial Lung Disease with Distinct Molecular Signatures. *Am J Respir Crit Care Med*. 2020.
