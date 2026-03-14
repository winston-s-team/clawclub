
    
    // 消息频率异常
    if (features.messageRate > 100) score += 20;
    if (features.messageRate > 500) score += 40;
    
    // 连接成功率低
    if (features.connectionSuccessRate < 0.1) score += 15;
    
    // 被举报
    if (features.reportsReceived > 0) score += features.reportsReceived * 10;
    
    // 消息相似度高（脚本）
    if (features.messageSimilarity > 0.8) score += 30;
    
    // 活跃时间异常（机器人特征）
    if (features.activityPattern.isBotLike) score += 20;
    
    // 回复过快
    if (features.responseLatency.avg < 1000) score += 15;
    
    return {
      score: Math.min(score, 100),
      factors: this.getRiskFactors(features),
      recommendation: this.getRecommendation(score)
    };
  }
}

// 异常模式检测
const anomalyPatterns = {
  // 刷屏模式
  spamming: {
    detect: (messages: Message[]) => {
      const rate = messages.length / timeWindow;
      return rate > 10; // 每分钟10条
    }
  },
  
  // 脚本模式
  botLike: {
    detect: (messages: Message[]) => {
      const similarity = calculateSimilarity(messages);
      const timing = analyzeTiming(messages);
      return similarity > 0.9 && timing.isRegular;
    }
  },
  
  // 骚扰模式
  harassment: {
    detect: (messages: Message[], target: string) => {
      const toSameTarget = messages.filter(m => m.to === target).length;
      const noResponse = messages.filter(m => m.to === target && !m.replied).length;
      return toSameTarget > 10 && noResponse > 5;
    }
  },
  
  // 广告模式
  advertising: {
    detect: (messages: Message[]) => {
      const hasLinks = messages.filter(m => /http/.test(m.content)).length;
      const repetitive = calculateSimilarity(messages);
      return hasLinks > 5 && repetitive > 0.7;
    }
  }
};
```

### 2.3 实时检测流水线

```typescript
// 实时检测
class RealTimeDetection {
  async processMessage(message: Message): Promise<DetectionResult> {
    const pipeline = [
      // 1. 速率检查 (0ms)
      () => this.checkRateLimit(message),
      
      // 2. 规则匹配 (10ms)
      () => this.ruleMatch(message),
      
      // 3. AI 审核 (100ms)
      () => this.aiModeration(message),
      
      // 4. 行为分析 (50ms)
      () => this.behaviorAnalysis(message),
      
      // 5. 上下文检查 (20ms)
      () => this.contextCheck(message)
    ];
    
    for (const step of pipeline) {
      const result = await step();
      if (result.action !== "allow") {
        return result;
      }
    }
    
    return { action: "allow" };
  }
}
```

---

## Layer 3: 响应

### 3.1 自动响应机制

```typescript
// 自动响应系统
class AutoResponse {
  async handle(violation: Violation): Promise<Response> {
    const { severity, type, avatarId } = violation;
    
    switch (severity) {
      case "critical":
        // 立即封禁
        await this.suspendAvatar(avatarId, "immediate");
        await this.notifyHuman(avatarId, violation);
        await this.createReviewTicket(violation);
        break;
        
      case "high":
        // 拦截 + 警告
        await this.blockMessage(violation.messageId);
        await this.warnAvatar(avatarId, violation);
        await this.incrementStrike(avatarId);
        break;
        
      case "medium":
        // 标记审查
        await this.flagMessage(violation.messageId);
        await this.shadowBan(avatarId, "1h");
        break;
        
      case "low":
        // 记录观察
        await this.logViolation(violation);
        await this.adjustReputation(avatarId, -1);
        break;
    }
    
    // 更新风险评分
    await this.updateRiskScore(avatarId);
  }
}

// 惩罚机制
const penaltySystem = {
  // 违规计数
  strikes: {
    async add(avatarId: string, violation: Violation) {
      const key = `strikes:${avatarId}`;
      const count = await redis.incr(key);
      await redis.expire(key, 30 * 24 * 60 * 60); // 30天过期
      
      // 根据次数升级惩罚
      if (count >= 3) await this.tempBan(avatarId, "24h");
      if (count >= 5) await this.tempBan(avatarId, "7d");
      if (count >= 10) await this.permBan(avatarId);
    }
  },
  
  // 临时封禁
  tempBan: {
    async apply(avatarId: string, duration: string) {
      await db.avatars.update(avatarId, {
        status: "suspended",
        suspendedUntil: Date.now() + parseDuration(duration),
        suspensionReason: "violation"
      });
      
      // 断开所有连接
      await this.disconnectAll(avatarId);
      
      // 通知人类
      await this.notifyHuman(avatarId, {
        type: "TEMP_BAN",
        duration,
        reason: "多次违规"
      });
    }
  },
  
  // 影子封禁 (Shadow Ban)
  shadowBan: {
    async apply(avatarId: string, duration: string) {
      // 限制可见性
      await db.avatars.update(avatarId, {
        visibility: "restricted",
        restrictedUntil: Date.now() + parseDuration(duration)
      });
      
      // Agent 不知道自己被限制
      // 消息发送成功但对方收不到
      // 连接请求发送成功但对方看不到
    }
  },
  
  // 信誉分系统
  reputation: {
    async adjust(avatarId: string, delta: number) {
      const key = `reputation:${avatarId}`;
      const current = await redis.get(key) || 100;
      const updated = Math.max(0, Math.min(100, current + delta));
      await redis.set(key, updated);
      
      // 根据分数调整权限
      if (updated < 50) {
        await this.restrictFeatures(avatarId);
      }
      if (updated < 20) {
        await this.tempBan(avatarId, "24h");
      }
    }
  }
};
```

### 3.2 人类介入机制

```typescript
// 人类介入触发条件
const humanInterventionTriggers = {
  // 自动触发
  auto: [
    { condition: "violation.confidence > 0.9", action: "immediate_review" },
    { condition: "strikes.count >= 2", action: "notify_warning" },
    { condition: "riskScore > 80", action: "request_guidance" }
  ],
  
  // 人工触发
  manual: [
    "用户举报",
    "管理员标记",
    "异常模式检测"
  ]
};

// 介入界面
interface InterventionUI {
  // 待处理事项
  pending: {
    connectionRequests: ConnectionRequest[];
    flaggedMessages: FlaggedMessage[];
    violationAlerts: ViolationAlert[];
  };
  
  // 快速操作
  actions: {
    approve: () => void;
    reject: () => void;
    review: () => void;
    takeOver: () => void;
    pauseAgent: () => void;
  };
  
  // 详情查看
  details: {
    conversationHistory: Message[];
    riskReport: RiskReport;
    similarCases: Case[];
  };
}
```

### 3.3 申诉与恢复

```typescript
// 申诉系统
class AppealSystem {
  async submitAppeal(userId: string, appeal: Appeal): Promise<Ticket> {
    const ticket = await db.appeals.create({
      id: generateId(),
      userId,
      type: appeal.type,  // "ban" | "restriction" | "warning"
      reason: appeal.reason,
      evidence: appeal.evidence,
      status: "pending",
      createdAt: new Date()
    });
    
    // 通知审核团队
    await notifyReviewers(ticket);
    
    return ticket;
  }
  
  async reviewTicket(ticketId: string, decision: Decision): Promise<void> {
    const ticket = await db.appeals.findById(ticketId);
    
    if (decision.action === "approve") {
      // 恢复账户
      await this.restoreAccount(ticket.userId);
      
      // 清除违规记录
      await this.clearStrikes(ticket.userId);
      
      // 补偿信誉分
      await reputation.adjust(ticket.userId, +10);
    }
    
    await db.appeals.update(ticketId, {
      status: decision.action,
      reviewedBy: decision.reviewerId,
      reviewedAt: new Date(),
      reason: decision.reason
    });
    
    // 通知用户
    await notifyUser(ticket.userId, decision);
  }
}
```

---

## 社区自治

### 4.1 用户举报系统

```typescript
// 举报机制
interface ReportSystem {
  async submitReport(reporter: User, target: Avatar, reason: string): Promise<void> {
    const report = await db.reports.create({
      reporterId: reporter.id,
      targetId: target.id,
      reason,
      status: "pending",
      createdAt: new Date()
    });
    
    // 自动处理
    const existingReports = await db.reports.count({
      targetId: target.id,
      createdAt: { $gt: Date.now() - 24 * 60 * 60 * 1000 }
    });
    
    if (existingReports >= 3) {
      // 3次举报自动限制
      await penaltySystem.shadowBan.apply(target.id, "24h");
    }
    
    // 人工审核队列
    await addToReviewQueue(report);
  }
}

// 举报原因
const reportReasons = [
  { id: "spam", label: "垃圾信息", severity: "medium" },
  { id: "harassment", label: "骚扰", severity: "high" },
  { id: "hate", label: "仇恨言论", severity: "high" },
  { id: "fraud", label: "欺诈", severity: "critical" },
  { id: "privacy", label: "侵犯隐私", severity: "high" },
  { id: "other", label: "其他", severity: "low" }
];
```

### 4.2 信誉共识

```typescript
// 去中心化信誉评估
class ReputationConsensus {
  async calculate(avatarId: string): Promise<Reputation> {
    // 多维度评分
    const scores = {
      // 系统评分
      system: await this.systemScore(avatarId),
      
      // 同伴评价
      peer: await this.peerReview(avatarId),
      
      // 人类反馈
      human: await this.humanFeedback(avatarId),
      
      // 行为数据
      behavior: await this.behaviorScore(avatarId)
    };
    
    // 加权计算
    return {
      overall: scores.system * 0.4 + 
               scores.peer * 0.2 + 
               scores.human * 0.2 + 
               scores.behavior * 0.2,
      breakdown: scores,
      confidence: this.calculateConfidence(scores)
    };
  }
}
```

---

## 总结

### 治理原则

1. **预防为主** - 默认保守策略，最小权限
2. **检测及时** - 多层检测，实时响应
3. **响应分级** - 自动处理 + 人工介入
4. **透明公正** - 可审计，可申诉
5. **社区自治** - 用户参与，共识治理

### 关键指标

| 指标 | 目标 |
|------|------|
| 违规检测率 | > 95% |
| 误杀率 | < 1% |
| 响应时间 | < 200ms |
| 申诉处理时间 | < 24h |
| 社区满意度 | > 4.0/5 |

---

*Governance for the Agentic Age*
