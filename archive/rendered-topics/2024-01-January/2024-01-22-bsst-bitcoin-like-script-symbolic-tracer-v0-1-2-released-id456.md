# B'SST: Bitcoin-like Script Symbolic Tracer v0.1.2 released

dgpv | 2024-01-22 17:42:11 UTC | #1

A new release of the most powerful analysis tool for Bitcoin and Elements scripts (yes, I want to brag! :smile:)

URL: https://github.com/dgpv/bsst/

New features and improvements: Plugins, assertions and assumptions, dynamic PICK/ROLL argument support, features to help with malleability analysis

More detailed description (Full release notes is at https://github.com/dgpv/bsst/blob/35143a90b6e6e1dcd61a2991a4881c04190c5569/release-notes.md):

* Rework plugin system, now plugins are 'general' in a sense that they can hook into different stages of analysis to observe or change various things. Also some useful plugins are added

* Add ability to set assertions on stack values and witnesses, and assumptions for data placeholders. Please see newly added "Assertions" and "Assumptions" sections in README

* Add ability to analyze opcodes that access the stack differently based on their arguments, like `PICK`, `ROLL`, `CHECKMULTISIG`, even if these arguments are not statically known

* Add ability to set aliases for witnesses with `// bsst-name-alias(wit<N>): alias_name` (where <N> is withess number). For example, aliased witnesses 0 will be shown in the report as `alias_name<wit0>`

* New setting: `--produce-model-values-for`. It is a set of glob patterns to specify which model values to produce. Please look at the help text for this setting for details. Data references now are not included in the default set to produce model values for, but can be enabled with `$*` pattern.

* More than one model value sample can be generated per analyzed value, if max number of samples is specified after `:` in the pattern given with `--produce-model-values-for` setting. The limitation of multiple samples is that samples are generated independently, that means for `$a $b ADD VERIFY` your can get `1, 0` as possible values for both `$a` and `$b`, even if they cannot be both 0 at the same time

* Byte sizes of model value samples will be shown in the report when `--report-model-value-sizes` is set to `true`

There was also other (less prominent) improvements and of course bugfixes.

A note on features to help with malleability analysis:

- Now that  report can contain sizes of the witnesses, it is easier to calculate total witness size.
- Model values are generated such that values with different sizes are generated first, so it is easier to find out all possible sizes.
- The model values that are the same in 'child' execution paths are now grouped together and is 'moved up' in the hierarchy of the execution paths in the report. It is now easier to see which values are the same despite branching.
- The sizes are 'moved up' execution path hierarchy independently of the values themselves. The report can show for example that a witness has the same size in all paths while model values might be different within the execution paths
-  Plugins that might be useful in analyzing malleability: `checksig_track_bsst_plugin.py` and `model_value_usage_track_bsst_plugin.py` (please look at `plugins/README.md` for details)

-------------------------

