1.	PS3: Precise Patch Presence Test based on Semantic Symbolic Signature, ICSE’24 (https://github.com/Qi-Zhan/ps3)  二进制比对，二进制函数符号缺失
2.	Towards Practical Binary Code Similarity Detection: Vulnerability Verification via Patch Semantic Analysis . TOSEM'23  (https://github.com/shouguoyang/Robin) 
3.	REACT: IR-Level Patch Presence Test for Binary ASE'24  （https://github.com/Qi-Zhan/React）  基于中间语言的比对，中间语言缺失

cve就是cve-2024-1086和cve-2022-0847





Patched:

- fffb0b52d5258554c645c966c6cbef7de50b851d

Unpatched:

- 85e068 9eb6b10cd3b2fb455d1b3f4d4d0b13ff78





Host jump-host
    HostName 192.168.104.61
    User username   zhangxb

​    IdentityFile ~/.ssh/id_rsa   

Host target-host
    HostName 192.168.158.101
    User zhangxb
    ProxyJump jump-host
    IdentityFile ~/.ssh/id_rsa   



ssh-copy-id zhangxb@192.168.104.61



内核漏洞补丁存在性检测步骤：

1. CVE选取：CVE-2023-38409

   - https://www.cve.org/CVERecord?id=CVE-2023-38409
   - Patch: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit?id=fffb0b52d5258554c645c966c6cbef7de50b851d
   - Commit: fffb0b52d5258554c645c966c6cbef7de50b851d
   - 更改函数：`drivers/video/fbdev/core/fbcon.c:set_con2fb_map`

2. linux内核编译：

   - menuconfig选项，确保开启：
     - Device Drivers > Graphics support > Frame buffer Devices > Support for frame buffer devices
     - Kernel hacking > Compile-time checks and compiler options > Debug information > 只要不选disable debug information就可以
   - 内核版本信息：
     - Patched commit : __fffb0b52d5258554c645c966c6cbef7de50b851d__
     - Unpatched commit : __85e068 9eb6b10cd3b2fb455d1b3f4d4d0b13ff78__

3. ps3输出：target为没有打补丁版本，检测result=vuln，结果正确

   > CVE-2023-38409
   > LinuxKernel_38409 truth is vuln
   > CVE-2023-38409 LinuxKernel_38409 truth = vuln result = vuln
   > linux CVE-2023-38409 (1.0, 1.0, 1.0)
   > RQ1 (1.0, 1.0, 1.0)





- command of Robin

  ```
  python main.py \
  	--mfi \
  	--cve_id CVE-2023-38409 \
  	--path_to_vul_bin ../binaries/CVE-2023-38409_fffb0b_vuln \
  	--path_to_patch_bin  ../binaries/CVE-2023-38409_fffb0b_patch \
    --vul_func_name set_con2fb_map
  ```

  ```
  # test
  python main.py --detect --cve_id CVE-2023-38409 --target_bin ../binaries/LinuxKernel_38409 --vul_func_name set_con2fb_map
  ```
  
  

```python
(pypy_angr) root@d201b62d0e01:/home/angr/Robin# python main.py --mfi --cve_id CVE-2023-38409 --path_to_vul_bin ../binaries/CVE-2023-38409_fffb0b_vuln --path_to_patch_bin  ../binaries/CVE-2023-38409_fffb0b_patch   --vul_func_name set_con2fb_map
WARNING | 2024-12-30 14:20:52,384 | cle.backends.elf.relocation | Unknown reloc 24 on AMD64
INFO    | 2024-12-30 14:20:58,703 | PoC_generation | MFI of CVE-2023-38409 Generating...
Traceback (most recent call last):
  File "main.py", line 62, in <module>
    gp.run(new_poc=True)
  File "/home/angr/Robin/MFI_operation/PoC_generation.py", line 74, in run
    self._pick_patch_block()
  File "/home/angr/Robin/MFI_operation/PoC_generation.py", line 105, in _pick_patch_block
    if self._input_generation_for_a_pair(check, guard):
  File "/home/angr/Robin/MFI_operation/PoC_generation.py", line 165, in _input_generation_for_a_pair
    patch_bin_project=self._patch_bin_project)
  File "/home/angr/Robin/patch_detection.py", line 838, in input_generation
    state = self.get_state_by_se(patch_bin_path, check_addr, patch_addr, patch_bin_project=patch_bin_project)
  File "/home/angr/Robin/patch_detection.py", line 343, in get_state_by_se
    cfg = angr_cfg_gen()
  File "/home/angr/Robin/patch_detection.py", line 330, in angr_cfg_gen
    cfg = self.patch_project.analyses.CFGFast()
  File "/root/.virtualenvs/pypy_angr/site-packages/angr/analyses/analysis.py", line 115, in __call__
    oself.__init__(*args, **kwargs)
  File "/root/.virtualenvs/pypy_angr/site-packages/angr/analyses/cfg/cfg_fast.py", line 664, in __init__
    self._analyze()
  File "/root/.virtualenvs/pypy_angr/site-packages/angr/analyses/forward_analysis/forward_analysis.py", line 226, in _analyze
    self._post_analysis()
  File "/root/.virtualenvs/pypy_angr/site-packages/angr/analyses/cfg/cfg_fast.py", line 1235, in _post_analysis
    self.make_functions()
  File "/root/.virtualenvs/pypy_angr/site-packages/angr/analyses/cfg/cfg_base.py", line 1330, in make_functions
    blockaddr_to_function
  File "/root/.virtualenvs/pypy_angr/site-packages/angr/analyses/cfg/cfg_base.py", line 1660, in _process_irrational_function_starts
    if block.vex.jumpkind not in ('Ijk_Boring', 'Ijk_InvalICache'):
  File "/root/.virtualenvs/pypy_angr/site-packages/angr/block.py", line 265, in vex
    cross_insn_opt=self._cross_insn_opt,
  File "/root/.virtualenvs/pypy_angr/site-packages/angr/engines/vex/lifter.py", line 228, in lift_vex
    raise SimEngineError("No bytes in memory for block starting at %#x." % addr)
angr.errors.SimEngineError: No bytes in memory for block starting at 0xffffffff81e5df0d.
```

