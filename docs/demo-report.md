# 演示报告

> 由 `make demo` 在本地 PostgreSQL 18 上实际执行生成；27/27 断言通过。


## 0. 报文契约注册

| 步骤 | 结果 | 说明 |
| --- | --- | --- |
| declaration/declaration | ✅ | registered v1 compat=strict fields=3 |
| scanner/scan | ✅ | registered v1 compat=backward fields=3 |
| screening/screening | ✅ | registered v1 compat=strict fields=4 |
| imaging/evidence | ✅ | registered v1 compat=backward fields=2 |
| review/manual_review | ✅ | registered v1 compat=strict fields=3 |
| stowage/container_stow | ✅ | registered v1 compat=backward fields=2 |
| manifest/flight_manifest | ✅ | registered v1 compat=backward fields=4 |
| custody/handover | ✅ | registered v1 compat=backward fields=3 |
| custody/package_change | ✅ | registered v1 compat=strict fields=4 |

## 1. 干净货件 AWB12345678901：暂定→正式分阶段发布

| 步骤 | 结果 | 说明 |
| --- | --- | --- |
| declaration | ✅ | 1 package declared for AWB 12345678901 |
| publish provisional @02:30 | ✅ | kind=provisional status=verified completeness=1.000 consumable=false |
| publish formal @03:00 | ✅ | kind=formal status=verified completeness=1.000 consumable=true diff=v1->v2, 3 new inputs |
| as-of publication 02:35 read | ✅ | v1 provisional status=verified |
| publish container AKE12345MU | ✅ | status=verified derived_from=1 shipment(s) |
| publish leg CA1234/2026-11-17/PEK-SHA | ✅ | status=verified derived_from=1 shipment(s) |
| replay a484afa6 | ✅ | published=verified recomputed=verified match=true |
| replay 09641d59 | ✅ | published=verified recomputed=verified match=true |

## 2. 重复补传 + 拆包旧证据 AWB12345678902

| 步骤 | 结果 | 说明 |
| --- | --- | --- |
| identical retransmission | ✅ | outcome=duplicate_identical duplicate_of=6d5b3ba4-c03d-4b81-bdd2-c769c6beae5b |
| conflicting retransmission | ✅ | outcome=duplicate_conflicting error=true (original 6d5b3ba4-c03d-4b81-bdd2-c769c6beae5b kept) |
| review correction | ✅ | review supersedes device verdict 6d5b3ba4 |
| split at 04:25 | ✅ | PKG-DUP-1 -> PKG-SPLIT-1A + PKG-SPLIT-1B (new composition version, not a mutation) |
| late scan citing retired package | ✅ | arrived 6 minutes after the split; cites PKG-DUP-1 which no longer exists as a version |
| children re-screened | ✅ | both split children carry their own screening verdict + evidence archive |
| publish split shipment v1 | ✅ | status=pending_evidence completeness=0.833 consumable=false |
| publish split shipment v2 (after re-declaration) | ✅ | status=verified completeness=0.983 diff=v1->v2, status pending_evidence -> verified, 1 new inputs |
| quality after split | ✅ | 1 cross-package/misaligned findings |
| stale scan disposition | ✅ | disposition=superseded_package; event arrived after the package was retired by a split/merge; evidence is misaligned and is not attributed to any child |
| composition versions | ✅ | 2 versions: [v1(PKG-DUP-1) v2(G-SPLIT-1A+G-SPLIT-1B)] |
| replay split v1 | ✅ | published=pending_evidence recomputed=pending_evidence |
| replay split v2 | ✅ | published=verified recomputed=verified |

## 3. 跨来源冲突 AWB65432100003：冲突待裁定而非代选

| 步骤 | 结果 | 说明 |
| --- | --- | --- |
| publish conflict shipment | ✅ | status=conflict_pending conflicts=1 consumable=false |
| wait_complete with open sources | ✅ | skipped=true reason=wait_complete: watermark not closed (missing sources [declaration manifest scanner screening]); no snapshot published |
| forced formal publish | ✅ | kind=formal status=conflict_pending consumable=false |

## 4. 舱单缺证据 AWB98765400004：不得签发确认状态，只输出质量告警与待补证

| 步骤 | 结果 | 说明 |
| --- | --- | --- |
| publish manifest-gap shipment | ✅ | status=pending_evidence consumable=false pending=[departure_handover_missing manifest_required_evidence_missing] |

## 5. 收尾校验：账本不可变 + 所有版本可复演 + 顶层约束

| 步骤 | 结果 | 说明 |
| --- | --- | --- |
| append-only guard | ✅ | UPDATE raw_events rejected by trigger: ERROR: table public.raw_events is append-only: UPDATE is not permitted (SQLSTATE P0001) |
| replay all snapshots | ✅ | 9/9 snapshots replay deterministically |
| status coverage | ✅ | map[conflict_pending:2 pending_evidence:2 verified:5] |

## 断言检查

| 检查 | 结果 | 依据 |
| --- | --- | --- |
| provisional published before sources closed | ✅ | early_provisional mode must publish at the cutoff even if sources are open |
| formal version follows provisional with explicit diff | ✅ | the second publication must be a new version superseding the provisional one |
| verified clean shipment is consumable-confirmed | ✅ | a formal verified snapshot is the only shape downstream may treat as confirmed |
| point-in-time query returns the then-readable provisional version | ✅ | as-of must reproduce what downstream could have read at that time |
| leg snapshot aggregates shipments across the leg | ✅ | the leg-level publication folds every shipment whose container flies that leg |
| clean shipment version replays deterministically | ✅ | every published version must be reproducible from its pinned inputs |
| clean shipment version replays deterministically | ✅ | every published version must be reproducible from its pinned inputs |
| identical retransmission suppressed as duplicate | ✅ | same idempotency content must not create a second event |
| conflicting retransmission quarantined, original retained | ✅ | a re-used idempotency key with new content must not overwrite |
| correction references the event it replaces | ✅ | supersedes relation is what makes the ruling auditable |
| split shipment does not silently inherit the retired parent's verdict | ✅ | the children were never covered by the original declaration, so v1 cannot be verified |
| re-declaration converges the split shipment in a new version | ✅ | post-split re-declaration is what makes the successor packages contributable |
| late old-package scan is misaligned, not auto-transferred | ✅ | a package retired by a split must not have its evidence silently attached to a child |
| stale scan appears in no child's evidence set | ✅ | the misaligned event must be retained for audit but absent from every successor's verdict inputs |
| split produced a new composition version | ✅ | the parent membership must be retired and the two children added as a new version |
| split shipment publication replays deterministically | ✅ | history including the split, the correction and the re-declaration must reproduce |
| split shipment publication replays deterministically | ✅ | history including the split, the correction and the re-declaration must reproduce |
| contradictory device results publish as conflict_pending | ✅ | without handover evidence the platform must not pick a verdict |
| conflict snapshot is not consumable-confirmed | ✅ | foreign systems must not treat a conflict as a confirmed status |
| wait_complete postpones publication until sources close | ✅ | the wait strategy deliberately holds the batch so no provisional value reaches downstream |
| even a forced formal snapshot keeps conflict_pending and is not consumable | ✅ | forcing the gate must never turn an unresolved conflict into a confirmed status |
| conflict snapshot replays deterministically | ✅ | the conflict set and decision must be reproducible |
| manifest citing missing required evidence cannot yield a confirmed status | ✅ | the snapshot must publish only quality warnings and pending-evidence items |
| manifest evidence gap is surfaced as a pending/blocking item | ✅ | a missing required evidence index must be visible to the operator |
| manifest-gap snapshot replays deterministically | ✅ | the refusal to confirm must be reproducible from history |
| ledger rejects overwrite | ✅ | UPDATE on raw_events is blocked by the DB |
| all published versions replay deterministically | ✅ | each downstream state must be reproducible by historical version |

**27/27 checks passed**
