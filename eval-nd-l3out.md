# Evaluation: ND L3Out object and native `cisco.nd` modules for the DC track

Branch `eval/nd-l3out`, started from tag `eval-nd-l3out-base` (`main` at `a944eb5`).
Nothing in this branch merges to `main`.

**Status: Phase 1 complete. Phase 2 (implementation, no pod) complete - see
section 5a. Phase 3 (pod tests) not started.**

Evidence labels used throughout:

- **Read from code**: read at the commit listed in section 2. Not tested.
- **Confirmed by test**: run on the pod. No Phase 1 item has this label.
- **Not verified**: not read and not tested. Do not rely on it.

Upstream references use the form `repo@commit:path:line`. `nd` means
`CiscoDevNet/ansible-nd`. `nac` means `netascode/ansible-dc-vxlan`. `dcnm` means
`CiscoDevNet/ansible-dcnm`.

## 1. Verdict (provisional, from Phase 1 only)

| Goal | Provisional verdict | Why |
| --- | --- | --- |
| 1. Replace VRF-Lite extension (stage 06) with the L3Out object | **Adopt after GA, if the pod passes** | The module and the API cover every lab value. Nothing upstream shows it working for this case: no test uses BGP with `configured_fabrics: fabric1`, an External fabric in Monitor Mode as fabric 2, or `attach`. `vrf2_name` and the interface-name form are open. All modules are unreleased (`version_added: 2.0.0`, `galaxy.yml` says 1.5.0). |
| 2. Replace `dcnm_rest` and older `cisco.dcnm` calls with `cisco.nd` modules | **Adopt partly, after GA** | 33 of 35 `dcnm_rest` calls have a module replacement or can be removed. The two recalculation calls (config-save and config-preview) do not: `config_actions` cannot run on its own, and `cisco.nd` has no config-preview endpoint. |

The final verdicts depend on Phase 3.

## 2. Upstream commits evaluated

| Repo | Ref | Commit | Date | Notes |
| --- | --- | --- | --- | --- |
| `CiscoDevNet/ansible-nd` | `develop` | `671e596` | 2026-11-12 | `galaxy.yml:5` `version: 1.5.0` |
| `CiscoDevNet/ansible-nd` | PR #629 `nd4x-nd_manage_l3out` | `bfa2111` | 2026-10-15 | Open. Only test tooling. No module code changes (see L3Out Q1). |
| `CiscoDevNet/ansible-nd` | PR #626 `nd4x-nd_manage_links` | `46fd7af` | 2026-11-06 | Open. Only test tooling under `plugins/`. |
| `CiscoDevNet/ansible-nd` | PR #657 `nd_manage_links_all` | n/a | updated 2026-10-27 | Open. Changes `nd_manage_links.py` and `orchestrators/links.py`. Does not change `ebgp_vrf_lite.py`. |
| `netascode/ansible-dc-vxlan` | `develop` | `e41c771` | 2026-10-29 | 0.9.1-dev. `galaxy.yml` depends on `cisco.dcnm >=3.13.1`. |
| `netascode/ansible-dc-vxlan` | tag `0.9.0` | n/a | n/a | The version the lab pins |
| `CiscoDevNet/ansible-dcnm` | tag `3.13.0` | `e00c486` | 2026-08-27 | The version the lab pins |
| `ansible/ansible` | tag `v2.17.14` | n/a | n/a | Persistent-connection socket key |

Repo recount on `main` `a944eb5` matches the brief: 35 `dcnm_rest` calls in stages
03 to 09 (3, 5, 4, 8, 2, 6, 7).

## 2a. Corrections and additions to the brief

1. `nd_manage_links`: `policy_type`, `template_name` and `template_inputs` sit under
   `config[].config_data`, not at the top of `config[]`
   (`nd@671e596:plugins/modules/nd_manage_links.py:120-170`).
2. `VrfExtensionModel` is not completely unused. `VrfAttachmentModel.extension_values`
   uses it (`nd@671e596:plugins/module_utils/models/manage_vrfs/vrf_attachment_models.py:350`),
   and `vrf_attachment_manager.py:669` builds that model. But the orchestrator has no
   reference to `extension`, so nothing a user enters can fill the field. The
   conclusion does not change: the current stage 06 method cannot move to `nd_manage_vrfs`.
3. **Safety:** `nd_manage_switches` defaults `config_actions` to `save: true`,
   `deploy: true` (`nd@671e596:plugins/module_utils/config_actions/policies.py:36-42`).
   Any task that adds the edge router must set both to `false`. Monitor Mode is a
   second safeguard but should not be the only one.
4. `nd_manage_switches` does not expose `max_hop` or `use_credential_for_write`. The
   model defaults are `maxHop: 2` and unset
   (`nd@671e596:plugins/module_utils/models/manage_switches/discovery_models.py:53,197-201`).
   Stage 04 sends `maxHop: 0` and `useCredentialForWrite: false`.
5. PR #629 changes no module code. Its `PARITY.md` says: "The original suite has no
   negative-validation, attach/detach, or destructive override scenario". So
   `attach` has no upstream test.
6. `nac` has 30 branches. The 31st `git branch -r` line is `HEAD`.
7. Persistent connections: see Platform Q1. Switching `ansible_network_os` per play
   does not by itself give `cisco.nd` its own connection.

## 3. Phase 1 answers

### L3Out

**Q1. Required fields for `subInterface` + `bgp` + `configured_fabrics: fabric1`.** Read from code.

- The module requires these for `merged`/`replaced`: `fabric1_name`, `fabric2_name`,
  `vrf1_name`, `vrf2_name`, `configured_fabrics`, `ip_version`,
  `connectivity_details`, `routing_details`
  (`nd@671e596:plugins/modules/nd_manage_l3out.py:859-868`, checked at 871-901 before
  any API call). `name` is required by the argspec.
- `vlan_id` is required for `subInterface`, range 1-4094
  (`.../models/l3out/l3out.py:240-243`).
- Each `switchN_details` needs `switch_id` and `interface_name` (argspec
  `l3out.py:353`). `switch_id` takes a serial or a management IP. An IP is
  resolved against `fabric1_name` / `fabric2_name` inventory
  (`orchestrators/manage_l3out.py:140-161`).
- `ipv4_mask_length` is documented as required for `ip_version: ipv4`. `bgp` is
  documented as required for `routing_protocol: bgp`
  (`nd_manage_l3out.py:159, 265-266`). Whether the model enforces these: Not verified.
- The peer ASN is `routing_details.bgp.fabric2_details.local_asn`. There is no
  neighbor-ASN field (`l3out.py:40-58`).
- `tenant1_name` is not enforced (see Q4).

Draft for PROD. IOT and MAIN differ only in name, VRF, `vlan_id` and addresses:

```yaml
- name: PROD-External            # 1-64 chars [A-Za-z0-9_-]
  fabric1_name: Pseudoco-DC1
  fabric2_name: External
  vrf1_name: PROD
  vrf2_name: "<open: see Q4>"
  configured_fabrics: fabric1
  ip_version: ipv4
  connectivity_details:
    routing_interface_type: subInterface
    links:
      - vlan_id: 2
        mtu: 1500
        ipv4_mask_length: 30
        switch1_details:
          switch_id: "<DC-Service-Leaf serial from dc_device_serials.yml>"
          interface_name: Ethernet1/8          # or Ethernet1/8.2, see Q1 conflict
          interface_ipv4_address: 192.168.252.5
        switch2_details:
          switch_id: 198.18.133.14             # resolved to serial in External
          interface_name: GigabitEthernet2     # or GigabitEthernet2.2
          interface_ipv4_address: 192.168.252.6
  routing_details:
    routing_protocol: bgp
    bgp:
      fabric1_details: { local_asn: "65000" }  # from global.nac.yaml
      fabric2_details: { local_asn: "65531" }  # dc_external_fabric_settings.BGP_AS
  attach: true
```

**Interface-name conflict (open).** Read from code.

- The module `EXAMPLES` use the sub-interface name (`Ethernet1/1.100`,
  `nd_manage_l3out.py:679,683,714,718,748,753`).
- The integration vars use the parent name with `vlan_id: 100`
  (`tests/integration/targets/nd_manage_l3out/vars/main.yaml:19-20,112-122`).
- The model passes `interface_name` through unchanged (`l3out.py:175-185`).
- PR #629's preflight GETs
  `/api/v1/manage/fabrics/{f}/switches/{id}/interfaces/{interface_name}` before
  creating the L3Out (`bfa2111:.../tasks/nd4x_demo_preflight.yaml:351`). A
  sub-interface cannot exist before the L3Out creates it, so that harness assumes
  the parent name.

What ND 4.2.1.10 accepts is Not verified. Pod plan: try the parent name first.

**Q2. Can every handoff value be pinned?** Read from code: yes, as far as the
payload goes. dot1q (`vlan_id`), both addresses, mask length, MTU and both ASNs are
payload fields, and the schema has nothing ND must allocate. Whether ND keeps the
values or overrides them: Not verified (Phase 3 step 3).

**Q3. Rejection of a Monitor Mode External fabric 2 or an IOS-XE fabric 2 device?**

- Module: no. Read from code. `manage_l3out.py` and `l3out.py` contain no check on
  fabric type, read-only/monitor state or platform.
- API: Not verified. The integration tests do not state the fabric 2 type or mode.
- One supporting detail: with `configured_fabrics: fabric1`, ND should generate no
  fabric 2 config. That is what Monitor Mode needs, but it does not show that ND
  accepts this combination.

**Q4. `tenant1_name` and `vrf2_name`.**

- `tenant1_name`:
  - Documented as "Only required if fabric 1 is being configured"
    (`nd_manage_l3out.py:94`). Not enforced.
  - The integration tests default to `omit` (`vars/main.yaml:11-12`). The PR #629
    harness checks tenant fields only when they are supplied
    (`nd4x_demo_preflight.yaml:14-17`).
  - Meaning for a VXLAN fabric: Not verified.
  - Plan: omit it, and ask the ND BU.
- `vrf2_name`:
  - Required even with `configured_fabrics: fabric1` (`nd_manage_l3out.py:859-868`).
  - The PR #629 preflight treats it as a VRF that must exist in fabric 2. It GETs
    `/api/v1/manage/fabrics/{fabric2}/vrfs/{vrf2_name}` and fails on non-200
    (`nd4x_demo_preflight.yaml:139-143,381`). That is the test author's assumption,
    not proof of ND behavior.
  - The External fabric has no VRF objects, and the router's VRFs are `10`, `101`
    and `102`.
  - Candidates:
    - (a) The router's real names `10`/`101`/`102`. This is the most accurate
      record.
    - (b) The fabric names `PROD`/`IOT`/`MAIN`. This matches what the controller
      stores today as `peer_vrf` (`dc_vrf_lite.yml`).
  - If ND requires the VRF to exist in fabric 2, neither candidate works without
    VRF objects in a Monitor Mode fabric. Not verified. This is the biggest open
    blocker.
  - Check mode cannot answer this, because no write reaches ND.

**Q5. Does `attach: true` deploy or only stage intent?** Read from code: it stages
intent only.

- `attach` POSTs `/api/v1/manage/l3OutActions/attach` with
  `{"attachments":[{"name","attach"}]}` and nothing else
  (`endpoints/v1/manage/manage_l3out.py:295-296`;
  `orchestrators/manage_l3out.py:338-415`). The module never calls configSave or
  deploy.
- The option text says "attached (deployed)" (`nd_manage_l3out.py:66`). The code
  does not do that.
- So a recalculate-and-deploy stage is still needed after it. Whether ND deploys on
  attach by itself: Not verified.
- In check mode with `attach` set, the module always reports `changed`
  (`nd_manage_l3out.py:925-926`). It also reports `changed` on every real run
  (line 943).
- Attach failures become warnings (lines 990-994). Some cases raise inside
  `attach_l3outs` (status ≥ 400, line 391), but `_handle_attachments` catches the
  exception and turns it into a warning too (lines 944-945).
- Phase 2 must therefore fail the task on any warning that matches
  `Attachment operation failed`, and read the result back.

**Q6. Does an L3Out reserve `topDownL3Dot1q`?**

- The module never calls the resource manager. Read from code: no resource
  reference in the L3Out module, orchestrator or model.
- Whether ND reserves dot1q 2/3/4 on the server side when it creates or attaches the
  L3Out: Not verified.
- `nd_manage_resource_manager` supports `topDownL3Dot1q` with scope
  `device_interface`
  (`models/manage_resource_manager/constants.py:49-50`), so it is the fallback.
- Pod plan: create the L3Outs without it, then run
  `nd_manage_resource_manager state: gathered` and see what ND recorded.
- Risk: dot1q left reserved from earlier VRF-Lite runs (the "Id[3] is already
  allocated to MAIN" case in `dc_vrf_lite.yml`) could block the create. On a clean
  pod this cannot happen. Cleanup must release them.

**Q7. `VRF_LITE_AUTOCONFIG: Manual`.**

- Keep it. It is the ND default anyway (`nd_manage_fabric_ibgp_vxlan.py:1287-1294`,
  `default: manual`).
- Under `Back2Back&ToExternal`, ND auto-creates an IFC and allocates dot1q values
  in its own order. That would still collide with the fixed tags.
- `DCI_SUBNET_RANGE` / `DCI_SUBNET_TARGET_MASK` only feed IFC default-IP allocation.
  With L3Out every address is pinned, so they may stop mattering. Not verified.
  Keep them until the pod shows they can go.
- The NaC model cannot express these keys for a VXLAN fabric. In `nac@0.9.0` they
  appear only in the MSD/MCFG templates
  (`roles/dtc/common/templates/ndfc_fabric/{msd,mcfg}_fabric/resources/*.j2:4-5`).
  So stage 03 keeps them.

**Q8. Stage 02 `dcnm_vrf state: replaced` vs an attached L3Out.**

- Confirmed in `dcnm@3.13.0:plugins/modules/dcnm_vrf.py:1544-1550`. When ND has
  `extensionValues` and the task has none, `replaced` sets `found = False` and
  pushes the attachment again. `nac@0.9.0` uses `state: replaced`
  (`roles/dtc/create/tasks/common_vxlan/vrfs_networks.yml:64,99`).
- If an attached L3Out does not appear in the VRF attachment's `extensionValues`,
  stage 02 cannot see it and there is no conflict. The "stage 06 must follow
  stage 02" rule could then go.
- If it does appear, every stage 02 re-run would try to strip it.
- Not verified. Phase 3 steps 5 and 8 answer it.

**Q9. Routing policy.** Read from code: omit `ipv4_peering_details` on both sides
and create no route-map, prefix-list, ACL or community-list objects. Zero
routing-policy modules are needed. All route-map fields are optional (`l3out.py:26-45`).
Whether ND's L3Out template adds route-maps of its own: Not verified. Compare
`show running-config bgp` with the baseline (Phase 3 step 3).

### Links

**Q1. Policy type, and mapping from the current `dcnm_links` task.** Read from code.

- Use `config_data.policy_type: ebgpVrfLite`. `template_name` applies only to
  `userDefined` (`nd_manage_links.py:158-161`).
- Which link type ND requires under a `subInterface` L3Out (`ebgpVrfLite`,
  `layer3DciVrfLite` or a plain link) is not stated anywhere in `cisco.nd`.
  `nd_manage_l3out` only says links are managed by another module (line 137). Not
  verified. This is a question for the ND BU.

| `04_external_fabric.yml:381-402` (`dcnm_links`) | `nd_manage_links` |
| --- | --- |
| `src_fabric` | `fabric_name` and `config[].src_fabric_name` |
| `dst_fabric` | `config[].dst_fabric_name` |
| `src_device` | `config[].src_switch_name` |
| `dst_device` | `config[].dst_switch_name` (or `dst_switch_ip: 198.18.133.14`) |
| `src_interface` / `dst_interface` | `config[].src_interface_name` / `dst_interface_name` |
| `template: ext_fabric_setup` | `config_data.policy_type: ebgpVrfLite` |
| `profile.ipv4_subnet` | `template_inputs.src_ip_address_mask` |
| `profile.neighbor_ip` | `template_inputs.dst_ip_address` |
| `profile.src_asn` | `template_inputs.src_ebgp_asn` (required, `ebgp_vrf_lite.py:33`) |
| `profile.dst_asn` | `template_inputs.dst_ebgp_asn` (required, line 34) |
| `profile.mtu` | `template_inputs.link_mtu` (line 41) |
| `profile.auto_deploy: false` | `auto_gen_config_peer: false`, plus `auto_gen_config_default_vrf: false`, `auto_gen_config_nx_peer_default_vrf: false` (lines 44-46), all set explicitly |

`ext_fabric_setup` does not appear anywhere in `nd@671e596`. Whether `ebgpVrfLite`
gives the same link that stage 04 creates today: Not verified.

Do not use `state: overridden`. It deletes links in scope that are not listed
(`nd_manage_links.py:231`).

**Q2. Can the link be created with the far end in Monitor Mode and no router config?**

- The module has no check on Monitor Mode or External fabrics (no match in
  `orchestrators/links.py`). Read from code.
- `auto_gen_config_peer: false` is the documented switch for peer config.
- Whether ND accepts this: Not verified.

**Q3. Are the IFC default IP pair and MTU still needed?**

- In `ebgpVrfLite` only the two ASNs are required. `src_ip_address_mask` and
  `dst_ip_address` are optional (`ebgp_vrf_lite.py:36-38`). Read from code.
- An omitted `link_mtu` is sent as 9216 (`nd_manage_links.py:169`;
  `ebgp_vrf_lite.py:41`). The 1500-byte sub-interfaces would still fit under it,
  but the two ends of the wire would disagree, which `dc_vrf_lite.yml` explicitly
  avoids. **Keep `link_mtu: 1500`.**
- Dropping the IP pair is plausible but Not verified. Try it on the pod; if ND
  rejects it, keep the pair.

### REST replacement

**Q1. Replacement table.** See section 4.

**Q2. Can `config_actions` replace stages 05 and 07?** No. Read from code.

- The fabric modules run `config_actions` only when the task sent a change
  (`nd_manage_fabric_ibgp_vxlan.py:2383`; `nd_manage_fabric_external.py:1016`).
  Stages 05 and 07 have no fabric change to send.
- Deploy targets come from a fresh switch-list read of
  `additionalData.configSyncStatus != inSync`
  (`orchestrators/config_actions/mixin.py:322-369`). The read is not preceded by a
  config-preview. Stage 05's comments record that, without config-preview, a switch
  can stay `inSync` after config-save. If that holds on ND 4.2.1.10, a
  `config_actions` deploy could find nothing to deploy. Not verified.
- `type: switch` means every switch in the fabric that is not `inSync` after the
  save, not only switches this task changed (`mixin.py:181-193`;
  `backends/fabric.py:83-103`).
- `cisco.nd` has no config-preview endpoint (no match under
  `plugins/module_utils/endpoints`).

Recommendation: keep one recalculate-and-deploy stage, built from NaC's `dtc.deploy`
role plus the two recalculation calls.

**Q3. Can `nd_manage_vrfs` replace anything in stages 06, 08 or 09?**

- Stage 06: no. The module has no extension fields (Correction 2), and L3Out
  replaces stage 06 anyway.
- Stage 08: `state: gathered` can replace the top-down VRF and attachment reads.
  The output shape differs, so the report logic changes. Whether `gathered` includes
  attachments: Not verified.
- Stage 09: `state: deleted` with an empty `config` removes all VRFs
  (`nd_manage_vrfs.py:41`). `nd_manage_networks` behaves the same way (line 55).
  Whether either detaches and deploys first, as `dcnm_vrf deleted` does: Not
  verified.
- Stage 02 keeps owning VRF creation through NaC.

### Platform

**Q1. Network OS and connection sharing.**

- `cisco.nd` needs `ansible_network_os: cisco.nd.nd` (`nd@671e596:README.md:122`)
  with the `cisco.nd.nd` httpapi plugin (`plugins/httpapi/nd.py`).
- One host cannot hold both values at once. Per-play `vars:` beat `group_vars`, but
  that does not separate the connections. In ansible-core 2.17.14 the
  persistent-connection socket key is
  `(remote_addr, port, remote_user, connection, playbook_pid)`
  (`ansible@v2.17.14:lib/ansible/cli/scripts/ansible_connection_cli_stub.py:268`).
  `ansible_network_os` is not part of it.
- Inside one `01_dc_deploy.yml` run, a `cisco.nd` play can therefore reuse stage
  02's live `cisco.dcnm` httpapi socket. `ansible_connect_timeout: 1000` keeps that
  socket alive for up to 1000 s idle.
- Options for Phase 2:
  - (a) A separate inventory host for the `cisco.nd` plays, plus `meta: reset_connection`
    at each switch between collections.
  - (b) Run the `cisco.nd` stages as a separate `ansible-playbook` invocation, which
    gets a new pid.
- Whether `reset_connection` fully tears down an httpapi socket: Not verified.
  Confirm with `-vvvv` socket-path logging.

**Q2. Python and ansible-core.** Read from code.

- `requires_ansible: '>=2.16.0'` (`meta/runtime.yml:2`). The lab's 2.17.14 meets it.
- `requirements.txt` needs `pydantic==2.12.5`, `requests_toolbelt`, `jsonpath-ng`
  and `lxml`.
- None of these is pinned in `ansible-automation/requirements.txt`. PyPI metadata
  shows no transitive pydantic in `nac-validate 1.0.0`, `nac-yaml 1.0.0` or
  `catalystcentersdk 3.1.3.0.1`. `requests-toolbelt` already comes in via
  `catalystcentersdk`. So there is no version conflict, but pydantic is a new
  dependency, and installing it needs your approval.
- The models use `str | None` annotations, which pydantic evaluates, for example
  `ebgp_vrf_lite.py:33`. That implies Python ≥ 3.10.
- The script server's Python version: Not verified. It runs `/usr/bin/python3`
  (`00_scriptserver_bootstrap/inventory/group_vars/script_servers/vars.yml:8`).
- Recommendation: use a virtualenv for the DC track only, so other tracks stay
  untouched.

**Q3. Check mode and diff.** Read from code. All modules in scope declare
`supports_check_mode=True`: `nd_manage_l3out`, `nd_manage_links`,
`nd_manage_resource_manager`, `nd_manage_fabric_ibgp_vxlan`,
`nd_manage_fabric_external`, `nd_manage_switches`, `nd_manage_vrfs`,
`nd_manage_networks`, `nd_manage_route_map` and `nd_rest`.

The state-machine modules return `before`, `after` and `diff` through
`module_utils/nd_output.py`. `nd_rest` returns no diff.

Exceptions:

- `nd_manage_l3out` reports `changed` in check mode whenever `attach` is set (Q5).
- `nd_rest` in check mode does not send the request (`nd_rest.py:292`).

## 4. REST and `cisco.dcnm` replacement table (proposed; final status after Phase 3)

Line numbers are from `main` `a944eb5`. The "Proposed" column holds one of these
values:

- **R**: replace with a module.
- **X**: remove. The module is idempotent or performs the check itself.
- **K**: keep as REST, with the reason shown.

| Stage:line | Purpose | Endpoint / module | Replacement | Proposed |
| --- | --- | --- | --- | --- |
| 03:56 | Fabric exists? | GET `lan-fabric/rest/control/fabrics/{f}` | `nd_manage_fabric_ibgp_vxlan state: gathered` (`merged` would try to *create* a missing fabric) | R |
| 03:108 | VRF-Lite nvPairs | `dcnm_fabric merged` | `nd_manage_fabric_ibgp_vxlan merged` `management.vrf_lite_auto_config/vrf_lite_subnet_range/vrf_lite_subnet_target_mask` (ibgp:1287-1304). Partial merged allowed (ibgp:129; `nd_state_machine.py:570-574`) | R |
| 03:138 | Read ND fabric object | GET `/api/v1/manage/fabrics/{f}` | Not needed: merged diffs for itself | X |
| 03:179 | licenseTier, telemetry, location | PUT `/api/v1/manage/fabrics/{f}` | Same merged task: `license_tier`, `telemetry_collection`, `location` (ibgp:49-83) | R |
| 04:105 | List fabrics | GET `.../control/fabrics` | `nd_manage_fabric_external` is idempotent | X |
| 04:136 | Create External, read-only | `dcnm_fabric merged` | `nd_manage_fabric_external merged` `management.bgp_asn`, `management.monitored_mode: true` (external:121,331) | R |
| 04:162 | Router present? | GET `/api/v1/manage/fabrics/{f}/switches` | `nd_manage_switches` is idempotent on `seed_ip` (switches:287-290) | X |
| 04:200 | Shallow discovery (IOS-XE) | POST `.../actions/shallowDiscovery` | `nd_manage_switches merged` `platform_type: ios-xe`, `preserve_config: true`, `config_actions: {save: false, deploy: false}` | R |
| 04:266 | Add router | POST `.../switches` | Same task. Caveats: `max_hop`/`useCredentialForWrite` not exposed (Correction 4) | R |
| 04:301 | Poll for router | GET `.../switches` | Module handles it (Not verified) | X |
| 04:(assert) | Serial matches stage 00 | n/a | Keep as a read-back assertion on `state: gathered` | keep check |
| 04:381 | Inter-fabric link | `dcnm_links merged ext_fabric_setup` | `nd_manage_links merged ebgpVrfLite` (Links Q1) | R |
| 05:53, 05:111 | Recalculate | POST `.../config-save` | No module. `config_actions` only after a change (REST Q2). Comment names `nd_manage_fabric_ibgp_vxlan.config_actions` | K |
| 05:59, 05:117 | Refresh compliance | GET `.../config-preview` | No endpoint in `cisco.nd` | K |
| 06:259 | Probe Manage API | GET `.../pools` | n/a | X |
| 06:284, 06:441 | Read / read back resources | GET `.../resources` | `nd_manage_l3out` only, or `nd_manage_resource_manager gathered` if needed (L3Out Q6) | X or R |
| 06:367 | Release stale dot1q | DELETE `.../resources/{id}` | `nd_manage_resource_manager deleted` if needed. Cleanup owns this. | X or R |
| 06:385 | Reserve dot1q | POST `.../resources` | `nd_manage_resource_manager merged` `pool_name: topDownL3Dot1q`, `scope_type: device_interface` only if ND does not reserve (Q6) | X or R |
| 06:493, 06:668 | Query / read back attachments | POST `.../vrfAttachments/query` | `nd_manage_l3out` `after` | X |
| 06:637 | Write extensions | POST `.../vrfAttachments` | `nd_manage_l3out merged` ×3 with `attach: true` | R |
| 07:44, 07:50 | Recalculate | config-save / config-preview | Merged into the single deploy stage | K (merged) |
| 08:362 | Inventory | GET `.../switchesByFabric` | `nd_manage_switches state: gathered` | R |
| 08:378, 08:420 | VRFs, VRF attachments | GET top-down `vrfs`, `vrfs/attachments` | `nd_manage_vrfs state: gathered` (attachments Not verified) | R |
| 08:396, 08:526 | Networks, attachments | GET top-down `networks`, `.../attachments` | `nd_manage_networks state: gathered` | R |
| 08:457 | VRF-Lite extensions | POST `vrfAttachments/query` | Check-mode `nd_manage_l3out` without `attach` → assert not changed. Attach status: `nd_rest` GET `/api/v1/manage/l3Outs?fabricName=` if the module output lacks it | R (+ K?) |
| 08:152 | BGP state on border leaf | `nxos_command` | Keep (design rule 4) | keep |
| 09:84, 09:233, 09:289 | List fabrics | GET `.../control/fabrics` | Modules are idempotent | X |
| 09:98, 09:108 | Delete networks / VRFs | `dcnm_network` / `dcnm_vrf deleted` | `nd_manage_networks` / `nd_manage_vrfs deleted` with empty `config` (detach-first Not verified) | R |
| 09:(new) | Detach, delete L3Outs | n/a | `nd_manage_l3out merged attach: false`, then `state: deleted` | new |
| 09:141 | Delete link | `dcnm_links deleted` | `nd_manage_links deleted` | R |
| 09:190, 09:198 | List and force-remove DC switches | GET `switchesByFabric`, DELETE `.../switches/{sn}?force=true` | `nd_manage_switches deleted` (force semantics Not verified) | R |
| 09:247, 09:255 | Same for External | same | `nd_manage_switches deleted` | R |
| 09:296 | Delete both fabrics | `dcnm_fabric deleted` | `nd_manage_fabric_ibgp_vxlan` / `nd_manage_fabric_external deleted` | R |
| 09:350-392 | Strip sub-interfaces on the leaf | `nxos_command` / `nxos_config` | Keep: switch-side, after the fabric is gone | keep |

Totals: 35 `dcnm_rest` calls today. Proposed: 2 kept (config-save and
config-preview, in the single deploy stage), plus at most 1 `nd_rest` read in
stage 08. All 5 non-NaC `cisco.dcnm` module calls are replaced.

Two smaller options outside this table:

- `nd_interface_subinterface_managed` exists upstream. It is not evaluated here.
- NaC `overlay_extensions.vrf_lites` applies only to External-fabric switches
  (`nac@develop:roles/validate/files/rules/external/502_policy_vrf_lite_cross_reference.py`).
  It does not apply here.

## 5. Proposed stage layout and estimates

| New | From | Content | Lines before → after (est.) | `dcnm_rest` before → after |
| --- | --- | --- | --- | --- |
| 00, 01, 02 | unchanged | Serial discovery, entry point, NaC create | unchanged | 0 → 0 |
| 03 fabric settings | 03 | `nd_manage_fabric_ibgp_vxlan` gathered + merged | 195 → ~70 | 3 → 0 |
| 04 external + link | 04 | `nd_manage_fabric_external`, `nd_manage_switches`, serial check, `nd_manage_links` | 412 → ~180 | 5 → 0 |
| 05 L3Out | 06 | 3× `nd_manage_l3out`, plus `nd_manage_resource_manager` if Q6 requires it | 768 → ~150 | 8 → 0 |
| 06 recalculate and deploy | 05 + 07 | Recalculation calls + `dtc.deploy` once | 120 + 63 → ~90 | 6 → 2 |
| 07 verify | 08 | gathered reads, L3Out drift check, SSH BGP check, report | 1246 → ~1000 | 6 → 0-1 |
| 08 cleanup | 09 | L3Out detach and delete, then the rest via `cisco.nd` | 420 → ~300 | 7 → 0 |
| **Total** | | | **3224 → ~1790** | **35 → 2-3** |

Collections: today `cisco.nac_dc_vxlan`, `cisco.dcnm`, `cisco.nxos`. After:
the same three plus `cisco.nd` from `develop`. Stage 02 and NaC's deploy role still
need `cisco.dcnm`.

Why one deploy instead of two, and the fallback:

- Two deploys exist today because ND builds the VRF-Lite extension prototype from
  the CDP adjacency on Ethernet1/8, and that adjacency appears only after the
  first deploy.
- The L3Out takes explicit interfaces, so it may not need the prototype. Not
  verified.
- If Phase 3 shows the L3Out create or attach still needs a deployed parent
  interface, keep the current pattern: deploy (stage 06a), L3Out (stage 05), then
  deploy again (stage 06b).

Ordering against stage 02: the L3Out stage keeps running after stage 02 until
Phase 3 answers Q8.

## 5a. Phase 2 implementation (code only, not run on a pod)

Approved on 2026-11-12: renumbered layout; Python dependencies installed into
the existing environment; `cisco.nd` stages run as separate `ansible-playbook`
processes; the pod may be torn down and rebuilt.

Installed: `cisco.nd` 1.5.0 from `CiscoDevNet/ansible-nd` commit `671e596`,
into the git-ignored `.eval_collections/` at the repository root, with
`ansible-galaxy collection install 'git+https://github.com/CiscoDevNet/ansible-nd.git,671e596' -p .eval_collections --no-deps`.
Only the DC track's `ansible.cfg` searches it (first). `pydantic` 2.12.5,
`requests_toolbelt`, `jsonpath-ng` and `lxml` installed with pip into the
existing local environment. Not installed on the script server yet.

### What was built

| Stage | File | Controller calls |
| --- | --- | --- |
| 01 | `01_dc_deploy.yml`, `tasks/run_stage.yml` | none; runs 02-07 as separate processes, logs each under `evidence/dc-deploy-<run>/` |
| 03 | `03_fabric_advanced_settings.yml` | `nd_manage_fabric_ibgp_vxlan` gathered + merged |
| 04 | `04_external_fabric.yml` | `nd_manage_fabric_external`, `nd_manage_switches` (`config_actions` save/deploy false), `nd_manage_links` `ebgpVrfLite`, 1 `nd_rest` read |
| 05 | `05_l3out.yml`, `tasks/l3out_config.yml` | `nd_manage_links` gathered (precheck), `nd_manage_l3out` merged, then merged with `attach: true` and `failed_when` on attach warnings |
| 06 | `06_recalculate_and_deploy.yml` | 2 `dcnm_rest` (config-save, config-preview), NaC `dtc.deploy` |
| 07 | `07_verify_fabric.yml` | `nd_manage_l3out` check mode, 6 `nd_rest` reads, SSH unchanged |
| 08 | `08_cleanup.yml` | `nd_manage_fabric_*` gathered/deleted, `nd_manage_l3out` (check-mode read, detach, delete), `nd_manage_networks`/`nd_manage_vrfs` deleted, `nd_manage_links` deleted, 2 `nd_rest` (switch list, forced remove) |

Variable files keep their names and the `dc_vrf_lite_*` prefix; keys that feed
a module were renamed to its options (`dc_vrf_lite_fabric_settings`,
`dc_vrf_lite_ifc`, `dc_advanced_nd_fabric_settings`,
`dc_external_fabric_settings`). Each handoff row gained an L3Out `name`.
Two per-run switches cover open items: `dc_vrf_lite_interface_form`
(`parent` | `subinterface`, B2) and the `peer_vrf` column (B1).

### Deviations from the Phase 1 table, and why

| Item | Phase 1 proposal | Built | Reason |
| --- | --- | --- | --- |
| Stage 07 inventory, VRF, network and attachment reads (5) | `gathered` modules | `nd_rest`, same endpoints | Read from code: `gathered` output is replay-safe config and drops `vrf_status` and attachment deploy state (`nd@671e596:plugins/module_utils/models/manage_vrfs/vrf_data_models.py:648-661`); the report is built on `lanAttachState`, serial and mode. Moved off `dcnm_rest` so the stage uses one collection per process |
| Stage 07 L3Out attach state | check-mode module | check-mode module + 1 `nd_rest` GET `/api/v1/manage/l3Outs?fabricName=`, saved as evidence | No module output carries an attach state (B4) |
| Stage 04 serial check | `gathered` | 1 `nd_rest` GET `/api/v1/manage/fabrics/External/switches` | `nd_manage_switches` outputs carry `seed_ip`, `role`, `platform_type` only (`nd_manage_switches.py:441-490`) |
| Stage 08 switch removal | `nd_manage_switches deleted` | `nd_rest` POST `.../switchActions/remove?force=true` + list | The module posts the same endpoint without `force` (`module_utils/manage_switches/nd_switch_resources.py:1278-1280`); the endpoint accepts it (`endpoints/v1/manage/manage_fabrics_switchactions.py:62,119-170`). `main` needs force for the border leaf |
| Stage 08 L3Out read | n/a | `nd_manage_l3out state: deleted` in check mode | The module has no `gathered`; check mode returns `before` without writing |

### Before and after (measured)

| | `main` `a944eb5` | `eval/nd-l3out` |
| --- | --- | --- |
| Stages 03-09 / 03-08 + shared tasks, lines | 3224 | 2553 (03 93, 04 245, 05 297, 06 91, 07 1250, 08 445, tasks 132) |
| `dcnm_rest` tasks | 35 | 2 (stage 06) |
| `nd_rest` tasks | 0 | 9 (04: 1, 07: 6, 08: 2) |
| `cisco.dcnm` module tasks outside NaC | 7 | 0 |
| Collections | `cisco.nac_dc_vxlan`, `cisco.dcnm`, `cisco.nxos` | the same + `cisco.nd` (develop) |

Stage 07 stayed near its old size: its reads moved transport, they did not
disappear. The Phase 1 estimate of ~1000 lines and 0-1 REST calls for it was
wrong for the reason in the first deviation row.

### Validation done without a pod

- `ansible-playbook --syntax-check` passes for all nine playbooks (local
  ansible-core 2.21.5).
- `ansible-lint` 26.10.1 passes at the `production` profile for the whole
  `playbooks/` tree, linted against the eval `cisco.nd`. ansible-lint puts
  `~/.ansible/collections` first, so a scratch collection path is needed to
  make it see the eval copy. Its `args` rule cannot check
  `nd_manage_links.template_inputs` (validated per policy at run time).
- The link payload and all three L3Out payloads, in both interface-name
  forms, were rendered from the real variable files and validated against the
  modules' own pydantic models (`NDLinkModel`, `L3OutModel`).
- The stage 07 L3Out comparison and both report templates were rendered with
  simulated controller output (matching, wrong MTU, missing).
- The stage runner was tested with dummy stages: extra-vars forwarding,
  change detection, stop on first failure, logs.

Found while building: the repo's `{% set %}`/`append`/`{{ out }}` idiom
returns a list on ansible-core 2.17 but a string on newer cores, which the
L3Out model rejects. The L3Out builder uses a single-expression `set_fact`
loop instead, which works on both. Other stages still use the idiom; they are
fine on the pinned 2.17.14.

Found in code review: `nd_manage_links` resolves both link endpoints in their
fabrics' inventories before any state runs, `deleted` included
(`nd@671e596:plugins/modules/nd_manage_links.py:839`;
`plugins/module_utils/orchestrators/links.py:89-90,124-150`). A delete
therefore fails when the router or a fabric is already gone. Stage 08 now
reads the links with `gathered` first and deletes only a link that exists;
stage 04 skips the link when the router is skipped. Worth raising upstream:
a delete should not need live endpoints.

Not verified (needs the pod): everything in section 8, plus that `nd_rest`
reaches the legacy `/appcenter/.../lan-fabric` read endpoints through the
`cisco.nd.nd` httpapi plugin, and that re-sending the controller's own L3Out
copy with `attach: false` is accepted (stage 08).

## 6. Phase 3 results

Not started.

## 7. Comparison, current method vs L3Out

To be completed after Phase 3. Static points already known:

- **Ordering:**
  - Today stage 06 must follow stage 02 on every pass (`dcnm_vrf replaced`).
  - With L3Out, that rule depends on Q8.
- **Idempotency:**
  - Today stage 06 carries its own drift comparison, read-back and 207 handling.
  - The L3Out module diffs for itself but reports `changed` on every run with
    `attach` set (Q5).
  - A second full run will therefore never show "no changes" unless the attach
    task is guarded. Phase 2 needs a read of the attach state to decide whether to
    attach.
- **Release status:** every `cisco.nd` module here is `version_added: 2.0.0` on
  `develop`. Nothing is in a released collection.

## 8. Blockers and risks

| # | Item | Evidence |
| --- | --- | --- |
| B1 | `vrf2_name` for an External fabric with no VRF objects. If ND requires a VRF object there, the L3Out cannot be created without one | Read from code (PR #629 preflight); ND behavior Not verified |
| B2 | Parent or sub-interface name in `interface_name` | Conflict read from code; Not verified |
| B3 | ND accepts BGP + `configured_fabrics: fabric1` with a Monitor Mode External fabric 2 and an IOS-XE peer | Not verified; no upstream test |
| B4 | `attach` is untested upstream, not idempotent, and reports failures as warnings | Read from code |
| B5 | All modules unreleased; PR #657 still changing `nd_manage_links` | Read from code |
| R1 | Persistent socket reuse between `cisco.dcnm` and `cisco.nd` in one run | Read from code (ansible-core) |
| R2 | `nd_manage_switches` would save and deploy by default | Read from code |
| R3 | `nd_manage_switches` discovers with `maxHop: 2`, where stage 04 uses 0. Whether only the seed is added | Not verified |
| R4 | `link_mtu` defaults to 9216 if omitted | Read from code |
| R5 | Stage 02 `replaced` stripping L3Out state (Q8) | Not verified |
| R6 | Leftover dot1q reservations blocking L3Out create (Q6) | Not verified |
| R7 | Python ≥ 3.10 and pydantic on the script server | Not verified |
| R8 | `nd_manage_links` sends documented defaults for omitted fields on update, which can reset values set outside the module | Read from code (`nd_manage_links.py:169`) |

## 9. Questions for the NaC developers and the ND BU

ND BU:

1. What do `vrf2_name` and `tenant2_name` mean when `configured_fabrics: fabric1` and
   fabric 2 is an External fabric in Monitor Mode with no VRF objects? Must the VRF
   exist?
2. For `subInterface`, is `interfaceName` the parent (`Ethernet1/8`) or the
   sub-interface (`Ethernet1/8.2`)?
3. Which inter-fabric link policy must exist under a `subInterface` L3Out:
   `ebgpVrfLite`, `layer3DciVrfLite` or another? Is the link's IP pair still used?
4. Does `l3OutActions/attach` deploy, or only stage until a fabric deploy?
5. Does the L3Out reserve `topDownL3Dot1q` itself? Does it show in the VRF
   attachment's `extensionValues`?
6. Does `VRF_LITE_AUTOCONFIG` affect L3Out behavior?
7. Is a GET for attach status planned in the module, so that `attach` can be
   idempotent?

NaC developers:

1. Issue #772 (`vxlan.routing_policies.l3out`) closed as completed on 2026-11-12, but
   none of the 30 branches has code for it. Where does that work live, and which
   collection will it use: `cisco.nd` or `cisco.dcnm`?
2. With `dcnm_vrf state: replaced`, will NaC leave L3Out-owned state on VRF
   attachments alone?
3. Is there a plan for NaC to depend on `cisco.nd` after `cisco.nd` 2.0.0 is
   released?

## 10. Merge plan

Not applicable until Phase 3.
