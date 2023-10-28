RS_boundedFaultReliability

A Jupyter notebook for some simple tests of the bounded-fault ECC mechanism which is relevant to DDR5 and HBM3 memory.

Using the very nice Reed-Solomon library by Tomer Fileba, whose work remains separate.

This notebook was triggered by reading Table V in the work "A Systematic Study of DDR4 DRAM Faults in the Field" by Beigi et al (https://ieeexplore.ieee.org/abstract/document/10071066).  That is an excellent study of actual faults in DDR4 DRAM deployed at scale, which is recommended reading for anyone working on DRAM reliability.

However, Table V at the end has two critical flaws:
- it assumes bounded fault behavior is the same in DDR5 as in DDR4, and ascribes a likely error rate to DDR5 although no DDR5 memory was used in the study.
- it neglects to note the important reduction in silent errors.

For the first issue, DDR5 has been designed to optimize the effectiveness of bounded fault detection.  Bounded faults are limited to the bits passing over 2 DQs (a DDR5 chip has 4DQs (half-duplex data lanes assigned each to a pin), so this is correction of up to half the data) or 32 bits of contiguous error pattern starting at half-chip boundaries.  The size of a bounded fault relevant to HBM3 is 16 bits.

These numbers were chosen to reflect the internal organization of DRAM so that classes of fault which fall within boundaries could be corrected, and these were expected to be a significant fraction of all multibit faults, especially after engineering review which could focus more resources on avoiding faults which are not bounded.  During the review process it was also discovered that the DDR4 chips from all vendors had been following a pattern of bit mapping to pins which was arbitrary but conflicted with the bounded fault principle.  In particular, contiguous bit patterns inside the DRAM will be mapped to all output pins, while the bounded fault principle requires that contiguous bits should map to a single pin.  This problem makes the bounded fault correction useless on DDR4, rendering the extrapolation of uncorrectable faults from DDR4 to DDR5 incorrect.  The mapping to single DQs is required in bounded fault so that both faults related to pins and board traces, and faults related to contiguous bits inside the array, are repairable by the bounded codes.

The elimination of silent errors is a significant benefit of using Reed-Solomon codes, which is available in the bounded fault scenario.

Bounded Fault operation is a cost saving measure in system design.  In DDR5 systems is allows a 10% reduction in the DRAM chip count while still correcting most errors and preventing silent errors.  It is important for the tradeoffs to be clearly understood before spending the extra money and power on chips which are not always needed.  Background can be found in "Improving Memory Reliability by Bounding DRAM Faults: DDR5 improved reliability features" by Criss et al (https://dl.acm.org/doi/abs/10.1145/3422575.3422803).

In order to evaluate the benefits this notebook constructs a simple model of Reed-Solomon correction of a 256-bit data transfer.  This size, corresponding to a half cacheline in most CPUs, is widely used in error correction for its effective balance of speed, size, and effectiveness.  In a bounded fault correction for DDR5 a 9th chip provides an extra 32 bits for Reed-Solomon check symbols.  This total size of 288 bits can be restated as bytes for an RS(36,32) bit code.  32 bytes of code, 4 bytes of check symbol.  It can correct any 2 bytes of contiguous symbol, which are on any 2 of the DQs.

The RS(36,32) may not be what every vendor uses, but it is a full strength algorithm and most likely several vendors use it, varying only in how cleverly they can implement the logic.  Tomer Fileba's library has a simple and clear implementation which is imported for this notebook.

The faults are mapped on a byte-per-DQ basis, following the design principles for DDR5 chips which support the JEDEC bounded fault proposal.  Fault models explore:
- the reliability of any 2 byte errors (all were corrected)
- the detection of whole-chip errors (about 1% are silent fails, likely under 0.05 FIT)
- the detection of multi-chip corruptions such as RowHammer might cause (also 1% silent)
- the quixotic exploration of what happens if you try erasure mapping to every chip to see if only one gets corrected (nope, RS will hallucinate bad corrections wherever you tell it to)
- a check of the RS(40,32) 10-chip mode detecting the worst case random multichip faults - it is flawless at detecting them.

In summary, bounded fault correction is not perfect, but it is a serious choice.  If you can recover from a detected fault - and most mission critical software can - then the residual risk of 0.05 silent failure is probably far lower than the silent failure mode in many other parts of the computer.  Those other fault sources are more serious and the money could be better directed.  See https://semiengineering.com/why-silent-data-errors-are-so-hard-to-find/ by Meixner for an introduction to the wider set of faults in real servers.  Arguably, the money and power spent on a 10th chip for "chipkill" support will be hard to assign value in most systems.