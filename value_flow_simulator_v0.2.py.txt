#!/usr/bin/env python3
"""
============================================================================
ACAA Cognitive Value Economy — Value Flow Simulation Engine v0.2
============================================================================
Track:      Research Only
Status:     CORRECTION RELEASE
Changes:    R1 Gini fix, R2 Attack engine, R3 Adaptive governance,
            R4 Multi-seed, R5 Regression suite
Seed Policy: Simulation seeds [42, 137, 256] independent of
             Reliability Analysis seed [20260808]
============================================================================
"""

import argparse
import hashlib
import json
import math
import random
import sys
from dataclasses import dataclass, field
from datetime import datetime, timezone
from enum import Enum
from typing import Dict, List, Optional, Tuple
from collections import defaultdict


class AgentType(Enum):
    HONEST = "Honest Producer"
    LAZY = "Lazy Producer"
    SYBIL = "Sybil Attacker"
    COLLUDER = "Colluding Ring"
    SPAMMER = "Spammer"
    CHALLENGER = "Challenger"
    GATEKEEPER = "Gatekeeper"
    ABSTAINER = "Strategic Abstainer"

class GateType(Enum):
    SCHEMA = "Schema Validation"
    CONTENT = "Content Validation"
    CROSS = "Cross-Artifact Check"
    REPRO = "Independent Reproduction"
    CRYPTO = "Cryptographic Integrity"

class Verdict(Enum):
    PASS = "PASS"
    FAIL = "FAIL"
    QUERY = "QUERY"

class CAULevel(Enum):
    L0 = "L0"
    L1 = "L1"
    L2 = "L2"
    L3 = "L3"

class ActionType(Enum):
    PRODUCTION = "Primary Production"
    VERIFICATION = "Verification"
    CHALLENGE = "Challenge"
    REPRODUCTION = "Reproduction"
    CORRECTION = "Correction"


@dataclass
class ProvenanceEvent:
    ts: int
    actor: str
    action: str
    detail: str

@dataclass
class CAURecord:
    cau_id: str
    actor_id: str
    action: ActionType
    verdict: Verdict
    quality: float
    complexity: float
    independence: float
    prov_score: float
    ts: int
    level: CAULevel
    negative: bool = False
    meta_depth: int = 0
    chain: List[ProvenanceEvent] = field(default_factory=list)

    @property
    def value(self) -> float:
        base = self.quality * self.complexity * self.independence * self.prov_score
        return -abs(base) if self.negative else base

    def add_prov(self, ev: ProvenanceEvent):
        self.chain.append(ev)

@dataclass
class Agent:
    aid: str
    atype: AgentType
    skill: float
    effort: float
    risk: float
    balance: float = 0.0
    produced: int = 0
    passed: int = 0
    failed: int = 0
    verified: int = 0
    challenges: int = 0
    challenges_won: int = 0
    active: bool = True
    sybil_parent: Optional[str] = None
    network: List[str] = field(default_factory=list)
    detected: bool = False
    isolated: bool = False

@dataclass
class GateCfg:
    gtype: GateType
    threshold: float
    cost: float
    noise: float

@dataclass
class SimParams:
    pi_p: float = 0.60
    pi_v: float = 0.20
    pi_s: float = 0.10
    pi_c: float = 0.10
    delta: float = 0.50
    lam: float = 1.50
    alpha_thr: float = 0.667
    periods: int = 100
    n_agents: int = 100
    noise: float = 0.05
    seed: int = 42

    @classmethod
    def from_json(cls, path: str) -> "SimParams":
        with open(path) as f:
            data = json.load(f)
        p = data["parameters"]
        e = data["execution"]
        return cls(
            pi_p=p["pi_p"]["value"], pi_v=p["pi_v"]["value"],
            pi_s=p["pi_s"]["value"], pi_c=p["pi_c"]["value"],
            delta=p["delta"]["value"], lam=p["lam"]["value"],
            alpha_thr=p["alpha_threshold"]["value"],
            periods=e["periods"], n_agents=e["num_agents"],
            noise=e["noise"], seed=e["seed"]
        )


class ValueFlowSim:
    def __init__(self, params: SimParams):
        self.p = params
        self.rng = random.Random(params.seed)
        self.agents: Dict[str, Agent] = {}
        self.ledger: List[CAURecord] = []
        self.gates: Dict[GateType, GateCfg] = {}
        self.history: List[Dict] = []
        self.period: int = 0
        self.cau_seq: int = 0
        self.stock: float = 0.0
        self.gate_cost: float = 0.0
        self.gate_value: float = 0.0
        self.attack_log: List[Dict] = []
        self.adaptive_log: List[Dict] = []
        self._init_gates()
        self._init_agents()

    def _init_gates(self):
        self.gates = {
            GateType.SCHEMA: GateCfg(GateType.SCHEMA, 0.5, 0.1, 0.02),
            GateType.CONTENT: GateCfg(GateType.CONTENT, 0.6, 0.3, 0.05),
            GateType.CROSS: GateCfg(GateType.CROSS, 0.65, 0.4, 0.05),
            GateType.REPRO: GateCfg(GateType.REPRO, 0.75, 0.8, 0.10),
            GateType.CRYPTO: GateCfg(GateType.CRYPTO, 0.5, 0.15, 0.01),
        }

    def _init_agents(self):
        dist = {AgentType.HONEST: 60, AgentType.LAZY: 10, AgentType.GATEKEEPER: 15,
                AgentType.CHALLENGER: 5, AgentType.ABSTAINER: 10}
        seq = 0
        for at, cnt in dist.items():
            for _ in range(cnt):
                seq += 1
                aid = f"A-{seq:04d}"
                skill = (self.rng.uniform(0.6, 1.0) if at == AgentType.HONEST else
                         self.rng.uniform(0.3, 0.6) if at == AgentType.LAZY else
                         self.rng.uniform(0.5, 0.9))
                self.agents[aid] = Agent(aid, at, skill,
                                         self.rng.uniform(0.3, 0.8),
                                         self.rng.uniform(0.3, 0.7))

    def _gini(self, vals: List[float]) -> float:
        clipped = [max(0.0, v) for v in vals]
        total = sum(clipped)
        if not clipped or total == 0:
            return 0.0
        s = sorted(clipped)
        n = len(s)
        weighted_sum = sum((i + 1) * x for i, x in enumerate(s))
        return (2.0 * weighted_sum) / (n * total) - (n + 1.0) / n

    def _quality(self, alpha: float) -> float:
        return 0.0 if alpha < self.p.alpha_thr else math.sqrt(alpha - self.p.alpha_thr)

    def _make_cau(self, agent: Agent, action: ActionType, verdict: Verdict,
                  q: float, cx: float, ind: float, neg: bool = False,
                  depth: int = 0) -> CAURecord:
        self.cau_seq += 1
        r = CAURecord(f"CAU-{self.cau_seq:06d}", agent.aid, action, verdict,
                      q, cx, ind, 1.0, self.period, CAULevel.L0, neg, depth)
        r.add_prov(ProvenanceEvent(self.period, agent.aid,
                                   f"CREATE:{action.value}", f"v={verdict.value}"))
        self.ledger.append(r)
        return r

    def _eval_gate(self, q: float, gt: GateType) -> Verdict:
        cfg = self.gates[gt]
        adj = q + self.rng.uniform(-cfg.noise, cfg.noise)
        if adj >= cfg.threshold: return Verdict.PASS
        if adj >= cfg.threshold * 0.8: return Verdict.QUERY
        return Verdict.FAIL

    def _allocate(self, val: float, producer: Agent, verifier: Optional[Agent], gt: GateType):
        producer.balance += val * self.p.pi_p
        if verifier: verifier.balance += val * self.p.pi_v
        self.stock += val * (self.p.pi_s + self.p.pi_c)
        self.gate_cost += self.gates[gt].cost
        self.gate_value += val

    def _detect_attacks(self):
        sybil_parents = defaultdict(int)
        collusion_rings = defaultdict(list)

        for agent in self.agents.values():
            if agent.atype == AgentType.SYBIL:
                sybil_parents[agent.sybil_parent] += 1
            elif agent.atype == AgentType.COLLUDER:
                ring_key = tuple(sorted(agent.network))
                collusion_rings[ring_key].append(agent.aid)

        for parent, count in sybil_parents.items():
            if count >= 5:
                for agent in self.agents.values():
                    if agent.sybil_parent == parent and not agent.detected:
                        agent.detected = True
                        agent.isolated = True
                        self.attack_log.append({
                            "period": self.period, "type": "sybil",
                            "action": "detected_and_isolated", "agents": [agent.aid]
                        })

        for ring_key, members in collusion_rings.items():
            if len(members) >= 5:
                for aid in members:
                    agent = self.agents.get(aid)
                    if agent and not agent.detected:
                        agent.detected = True
                        agent.isolated = True
                        self.attack_log.append({
                            "period": self.period, "type": "collusion",
                            "action": "detected_and_isolated", "agents": [aid]
                        })

        for agent in self.agents.values():
            if agent.atype == AgentType.SPAMMER and agent.produced > 20:
                fail_rate = agent.failed / max(agent.produced, 1)
                if fail_rate > 0.8 and not agent.detected:
                    agent.detected = True
                    agent.isolated = True
                    self.attack_log.append({
                        "period": self.period, "type": "spam",
                        "action": "detected_and_isolated", "agents": [agent.aid]
                    })

    def _adaptive_governance(self):
        if self.period < 10:
            return
        recent = self.history[-10:]
        avg_failure = sum(m["failure_rate"] for m in recent) / len(recent)
        avg_efficiency = sum(m["gate_efficiency"] for m in recent) / len(recent)
        gini = recent[-1]["gini_coefficient"] if recent else 0

        adjusted = False
        adjustments = {}

        if avg_failure > 0.5 and self.p.alpha_thr < 0.9:
            self.p.alpha_thr = min(0.9, self.p.alpha_thr + 0.05)
            adjustments["alpha_threshold"] = self.p.alpha_thr
            adjusted = True

        if avg_efficiency < 0.3 and self.p.pi_v > 0.05:
            self.p.pi_v = max(0.05, self.p.pi_v - 0.02)
            adjustments["pi_v"] = self.p.pi_v
            adjusted = True

        if gini > 0.8 and self.p.pi_s < 0.3:
            self.p.pi_s = min(0.3, self.p.pi_s + 0.02)
            adjustments["pi_s"] = self.p.pi_s
            adjusted = True

        if adjusted:
            self.adaptive_log.append({
                "period": self.period,
                "trigger_metrics": {
                    "avg_failure": round(avg_failure, 4),
                    "avg_efficiency": round(avg_efficiency, 4),
                    "gini": round(gini, 4)
                },
                "adjustments": adjustments
            })

    def _run_period(self):
        self.period += 1
        gks = [a for a in self.agents.values()
               if a.atype == AgentType.GATEKEEPER and a.active and not a.isolated]

        self._detect_attacks()

        for agent in list(self.agents.values()):
            if not agent.active or agent.isolated:
                continue

            if agent.atype in (AgentType.HONEST, AgentType.LAZY, AgentType.SPAMMER,
                                AgentType.SYBIL, AgentType.COLLUDER):
                effort = agent.effort * self.rng.uniform(0.5, 1.5)
                quality = min(1.0, agent.skill * effort)
                gt = self.rng.choice(list(GateType))
                agent.produced += 1

                if gks:
                    vk = self.rng.choice(gks)
                    vk.verified += 1
                    verdict = self._eval_gate(quality, gt)

                    if verdict == Verdict.PASS:
                        agent.passed += 1
                        q = self._quality(quality)
                        cau = self._make_cau(agent, ActionType.PRODUCTION, verdict, q, 1.0, 1.0)
                        cau.level = CAULevel.L1
                        self._allocate(cau.value, agent, vk, gt)
                    elif verdict == Verdict.FAIL:
                        agent.failed += 1
                        neg_q = self._quality(quality)
                        self._make_cau(agent, ActionType.PRODUCTION, verdict, neg_q, 1.0, 1.0, neg=True)
                        agent.balance -= neg_q * self.p.lam

            elif agent.atype == AgentType.CHALLENGER:
                targets = [c for c in self.ledger if c.level != CAULevel.L0 and not c.negative]
                if targets:
                    tgt = self.rng.choice(targets)
                    agent.challenges += 1
                    if self.rng.random() < agent.skill * (1.0 - tgt.quality):
                        agent.challenges_won += 1
                        c = self._make_cau(agent, ActionType.CHALLENGE, Verdict.PASS, 0.8, 1.2, 1.0)
                        agent.balance += c.value
                        tgt_actor = self.agents.get(tgt.actor_id)
                        if tgt_actor:
                            tgt_actor.balance -= c.value * self.p.lam

        for cau in self.ledger:
            if cau.meta_depth > 0:
                cau.quality *= self.p.delta ** cau.meta_depth

        for cau in self.ledger:
            if self.period - cau.ts > 50 and not cau.negative:
                cau.quality *= 0.95

        self._adaptive_governance()
        self._record()

    def _record(self):
        active = [a for a in self.agents.values() if a.active and not a.isolated]
        bals = [a.balance for a in active]
        total_sub = sum(a.produced for a in active)
        total_fail = sum(a.failed for a in active)
        total_ch = sum(a.challenges for a in active)
        won_ch = sum(a.challenges_won for a in active)
        detected_count = sum(1 for a in self.agents.values() if a.detected)
        isolated_count = sum(1 for a in self.agents.values() if a.isolated)

        self.history.append({
            "period": self.period,
            "total_cau_stock": round(self.stock, 6),
            "total_agent_balance": round(sum(bals), 6),
            "active_agents": len(active),
            "gini_coefficient": round(self._gini(bals), 6),
            "gate_efficiency": round(self.gate_value / max(self.gate_cost, 1e-9), 6),
            "failure_rate": round(total_fail / max(total_sub, 1), 6),
            "total_artifacts": total_sub,
            "total_failures": total_fail,
            "total_verifications": sum(a.verified for a in active),
            "total_challenges": total_ch,
            "challenge_success_rate": round(won_ch / max(total_ch, 1), 6),
            "detected_agents": detected_count,
            "isolated_agents": isolated_count,
        })

    def inject_sybil(self, n=20):
        base = len(self.agents) + 1
        pid = f"SYBIL-P-{base:04d}"
        for i in range(n):
            aid = f"SYBIL-{base+i:04d}"
            self.agents[aid] = Agent(aid, AgentType.SYBIL, self.rng.uniform(0.3,0.5),
                                     0.1, 0.9, sybil_parent=pid)

    def inject_collusion(self, n=10):
        base = len(self.agents) + 1
        ring = [f"COLLUDER-{base+i:04d}" for i in range(n)]
        for aid in ring:
            self.agents[aid] = Agent(aid, AgentType.COLLUDER, self.rng.uniform(0.4,0.6),
                                     0.2, 0.8, network=ring)

    def inject_spam(self, n=5):
        base = len(self.agents) + 1
        for i in range(n):
            aid = f"SPAM-{base+i:04d}"
            self.agents[aid] = Agent(aid, AgentType.SPAMMER, self.rng.uniform(0.1,0.3),
                                     0.05, 1.0)

    def inject_gate_inflation(self):
        for gt in self.gates:
            self.gates[gt].threshold = max(0.1, self.gates[gt].threshold - 0.3)

    def inject_strategic_abstention(self, n=15):
        base = len(self.agents) + 1
        for i in range(n):
            aid = f"ABSTAIN-{base+i:04d}"
            self.agents[aid] = Agent(aid, AgentType.ABSTAINER, self.rng.uniform(0.5,0.8),
                                     0.1, 0.1)

    def inject_negative_exploitation(self, n=5):
        base = len(self.agents) + 1
        for i in range(n):
            aid = f"EXPLOIT-{base+i:04d}"
            self.agents[aid] = Agent(aid, AgentType.CHALLENGER, self.rng.uniform(0.2,0.4),
                                     0.1, 0.9)

    def run(self, periods=None):
        for _ in range(periods or self.p.periods):
            self._run_period()
        return self.history

    def sensitivity(self, param: str, values: List[float]) -> List[Dict]:
        results = []
        for v in values:
            setattr(self.p, param, v)
            sim = ValueFlowSim(self.p)
            m = sim.run()
            f = m[-1] if m else {}
            results.append({"value": v, **{k: f.get(k) for k in
                ["total_cau_stock","gini_coefficient","gate_efficiency","failure_rate"]}})
        return results

    def export(self) -> Dict:
        return {
            "params": vars(self.p),
            "periods_executed": self.period,
            "cau_records": len(self.ledger),
            "agents": len(self.agents),
            "metrics": self.history,
            "attack_log": self.attack_log,
            "adaptive_log": self.adaptive_log,
            "provenance_events": sum(len(c.chain) for c in self.ledger),
        }


def run_regression_tests():
    print("=" * 60)
    print("REGRESSION TEST SUITE — Engine v0.2")
    print("=" * 60)
    passed = 0
    failed = 0

    sim = ValueFlowSim(SimParams(periods=1))

    gini_eq = sim._gini([1.0, 1.0, 1.0, 1.0])
    if abs(gini_eq) < 0.001:
        print("R5-T1: Gini perfect equality = 0.0 PASS")
        passed += 1
    else:
        print(f"R5-T1: Gini equality failed: {gini_eq} FAIL")
        failed += 1

    gini_neg = sim._gini([-1.0, 1.0, 1.0, 1.0])
    if 0 <= gini_neg <= 1:
        print("R5-T2: Gini negative clipping PASS")
        passed += 1
    else:
        print(f"R5-T2: Gini negative handling failed: {gini_neg} FAIL")
        failed += 1

    gini_known = sim._gini([1, 2, 3, 4])
    if 0.2 <= gini_known <= 0.3:
        print(f"R5-T3: Gini known value = {gini_known:.4f} PASS")
        passed += 1
    else:
        print(f"R5-T3: Gini known value failed: {gini_known} FAIL")
        failed += 1

    sim2 = ValueFlowSim(SimParams(periods=20, seed=42))
    sim2.inject_sybil(20)
    sim2.run()
    sybil_detected = sum(1 for a in sim2.agents.values()
                         if a.atype == AgentType.SYBIL and a.detected)
    if sybil_detected >= 15:
        print(f"R5-T4: Sybil detection = {sybil_detected}/20 PASS")
        passed += 1
    else:
        print(f"R5-T4: Sybil detection failed: {sybil_detected}/20 FAIL")
        failed += 1

    complete = sum(1 for c in sim2.ledger if len(c.chain) > 0)
    integrity = complete / max(len(sim2.ledger), 1) * 100
    if integrity >= 99.0:
        print(f"R5-T5: Provenance integrity = {integrity:.1f}% PASS")
        passed += 1
    else:
        print(f"R5-T5: Provenance integrity failed: {integrity:.1f}% FAIL")
        failed += 1

    print(f"\nRegression: {passed} PASS / {failed} FAIL")
    return failed == 0


def sha256_file(path: str) -> str:
    h = hashlib.sha256()
    with open(path, "rb") as f:
        for chunk in iter(lambda: f.read(8192), b""):
            h.update(chunk)
    return h.hexdigest()


def main():
    parser = argparse.ArgumentParser(description="ACAA Value Flow Simulator v0.2")
    parser.add_argument("--config", default="config_v2.0.json")
    parser.add_argument("--scenario", default="baseline",
                        choices=["baseline","sybil","collusion","spam",
                                 "gate_inflation","strategic_abstention",
                                 "negative_exploitation"])
    parser.add_argument("--sensitivity", type=str, default=None)
    parser.add_argument("--multi-seed", action="store_true")
    parser.add_argument("--regression", action="store_true")
    parser.add_argument("--output", default=None)
    args = parser.parse_args()

    if args.regression:
        success = run_regression_tests()
        sys.exit(0 if success else 1)

    params = SimParams.from_json(args.config)

    if args.multi_seed:
        seeds = [42, 137, 256]
        results = {}
        for seed in seeds:
            params.seed = seed
            sim = ValueFlowSim(params)
            sim.run()
            results[f"seed_{seed}"] = sim.export()
        with open(args.output or "multi_seed_results.json", "w") as f:
            json.dump(results, f, indent=2)
        print(f"Multi-seed execution complete: {seeds}")
        return

    sim = ValueFlowSim(params)

    if args.sensitivity:
        ranges = {
            "lam": [0.5, 1.0, 1.5, 2.0, 2.5, 3.0],
            "delta": [0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9],
            "pi_c": [0.05, 0.1, 0.15, 0.2, 0.25, 0.3],
            "alpha_threshold": [0.5, 0.55, 0.6, 0.65, 0.7, 0.75, 0.8, 0.85, 0.9],
            "pi_p": [0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9],
        }
        vals = ranges.get(args.sensitivity, [0.5, 1.0, 1.5, 2.0, 2.5, 3.0])
        result = {"param": args.sensitivity, "values": vals,
                  "results": sim.sensitivity(args.sensitivity, vals)}
    else:
        scenario_map = {
            "sybil": lambda: sim.inject_sybil(),
            "collusion": lambda: sim.inject_collusion(),
            "spam": lambda: sim.inject_spam(),
            "gate_inflation": lambda: sim.inject_gate_inflation(),
            "strategic_abstention": lambda: sim.inject_strategic_abstention(),
            "negative_exploitation": lambda: sim.inject_negative_exploitation(),
        }
        if args.scenario in scenario_map:
            scenario_map[args.scenario]()
        sim.run()
        result = sim.export()
        result["scenario"] = args.scenario

    result["execution_timestamp"] = datetime.now(timezone.utc).isoformat()
    result["random_seed"] = params.seed
    result["engine_hash"] = sha256_file(__file__) if __file__ else "N/A"

    output_path = args.output or f"{args.scenario}_results.json"
    with open(output_path, "w") as f:
        json.dump(result, f, indent=2, ensure_ascii=False)

    print(f"Execution complete. Results saved to: {output_path}")

if __name__ == "__main__":
    main()