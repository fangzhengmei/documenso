# 团队角色权限系统设计报告

## 一、角色定义与真实对应关系

### 1.1 系统中实际存在的角色类型

**组织与团队角色层级：

| 概念角色 | 系统真实对应 | 定义位置 | 说明 |
|---------|-------------|---------|------|
| 所有者 | `Organisation.ownerUserId` | `packages/prisma/schema.prisma:719 | 组织拥有者，独立于团队角色体系，具有组织最高权限 |
| 管理员 | `TeamMemberRole.ADMIN` | `packages/prisma/schema.prisma:822` | 团队管理员，拥有团队全部权限 |
| 经理 | `TeamMemberRole.MANAGER` | `packages/prisma/schema.prisma:823` | 团队经理，可管理团队设置和成员 |
| 成员 | `TeamMemberRole.MEMBER` | `packages/prisma/schema.prisma:824` | 团队普通成员，仅可见公开文档 |
| 文档参与者 | `RecipientRole.SIGNER/APPROVER` | `packages/prisma/schema.prisma:574` | 文档级角色，非团队角色 |
| 查看人 | `RecipientRole.VIEWER` | `packages/prisma/schema.prisma:576` | 文档级只读角色，非团队角色 |

> **重要说明**：
- "文档参与者"和"查看人"是文档级接收者角色（RecipientRole），不属于团队角色体系
- 团队成员（TeamMemberRole）控制团队级权限，接收者角色（RecipientRole）控制文档级操作权限

### 1.2 角色权限映射配置

**文件位置**：`packages/lib/constants/teams.ts:31-34`

```typescript
export const TEAM_MEMBER_ROLE_PERMISSIONS_MAP = {
  DELETE_TEAM: [TeamMemberRole.ADMIN],
  MANAGE_TEAM: [TeamMemberRole.ADMIN, TeamMemberRole.MANAGER],
} satisfies Record<string, TeamMemberRole[]>;
```

| 操作权限 | 允许角色 |
|---------|---------|
| DELETE_TEAM | 仅 ADMIN |
| MANAGE_TEAM | ADMIN, MANAGER |

### 1.3 角色层级关系

**文件位置**：`packages/lib/constants/teams.ts:48-52`

```typescript
export const TEAM_MEMBER_ROLE_HIERARCHY = {
  [TeamMemberRole.ADMIN]: [TeamMemberRole.ADMIN, TeamMemberRole.MANAGER, TeamMemberRole.MEMBER],
  [TeamMemberRole.MANAGER]: [TeamMemberRole.MANAGER, TeamMemberRole.MEMBER],
  [TeamMemberRole.MEMBER]: [TeamMemberRole.MEMBER],
} satisfies Record<TeamMemberRole, TeamMemberRole[]>;
```

| 当前角色 | 可管理的角色 |
|---------|-------------|
| ADMIN | ADMIN, MANAGER, MEMBER |
| MANAGER | MANAGER, MEMBER |
| MEMBER | MEMBER |

### 1.4 文档可见性映射

**文件位置**：`packages/lib/constants/teams.ts:36-40`

```typescript
export const TEAM_DOCUMENT_VISIBILITY_MAP = {
  [TeamMemberRole.ADMIN]: [DocumentVisibility.ADMIN, DocumentVisibility.MANAGER_AND_ABOVE, DocumentVisibility.EVERYONE],
  [TeamMemberRole.MANAGER]: [DocumentVisibility.MANAGER_AND_ABOVE, DocumentVisibility.EVERYONE],
  [TeamMemberRole.MEMBER]: [DocumentVisibility.EVERYONE],
} satisfies Record<TeamMemberRole, DocumentVisibility[]>;
```

| 团队角色 | 可见文档范围 |
|---------|-------------|
| ADMIN | EVERYONE, MANAGER_AND_ABOVE, ADMIN |
| MANAGER | EVERYONE, MANAGER_AND_ABOVE |
| MEMBER | EVERYONE |

---

## 二、权限断言工具函数

### 2.1 操作权限检查

**文件位置**：`packages/lib/utils/teams.ts:43-48`

```typescript
export const canExecuteTeamAction = (
  action: keyof typeof TEAM_MEMBER_ROLE_PERMISSIONS_MAP,
  role: keyof typeof TEAM_MEMBER_ROLE_MAP,
) => {
  return TEAM_MEMBER_ROLE_PERMISSIONS_MAP[action].some((i) => i === role);
};
```

**作用**：检查指定角色是否有权执行特定操作

**使用示例**：
```typescript
// 检查是否可以管理团队设置
if (canExecuteTeamAction('MANAGE_TEAM', team.currentTeamRole)) {
  // 显示设置入口
}
```

### 2.2 角色层级权限检查

**文件位置**：`packages/lib/utils/teams.ts:69-74`

```typescript
export const isTeamRoleWithinUserHierarchy = (
  currentUserRole: keyof typeof TEAM_MEMBER_ROLE_MAP,
  roleToCheck: keyof typeof TEAM_MEMBER_ROLE_MAP,
) => {
  return TEAM_MEMBER_ROLE_HIERARCHY[currentUserRole].some((i) => i === roleToCheck);
};
```

**作用**：检查当前用户角色是否有权管理目标角色（用于成员管理）

### 2.3 文档可见性检查

**文件位置**：`packages/lib/utils/teams.ts:57-59`

```typescript
export const canAccessTeamDocument = (role: TeamMemberRole, visibility: DocumentVisibility) => {
  return TEAM_DOCUMENT_VISIBILITY_MAP[role].some((i) => i === visibility);
};
```

### 2.4 获取用户最高角色

**文件位置**：`packages/lib/utils/teams.ts:76-89`

```typescript
export const getHighestTeamRoleInGroup = (groups: TeamGroup[]): TeamMemberRole => {
  let highestTeamRole: TeamMemberRole = LOWEST_TEAM_ROLE;

  groups.forEach((group) => {
    const currentRolePriority = TEAM_MEMBER_ROLE_HIERARCHY[group.teamRole].length;
    const highestTeamRolePriority = TEAM_MEMBER_ROLE_HIERARCHY[highestTeamRole].length;

    if (currentRolePriority > highestTeamRolePriority) {
      highestTeamRole = group.teamRole;
    }
  });

  return highestTeamRole;
};
```

**实现原理**：通过角色层级数组的长度来判断角色优先级，长度越大优先级越高。

### 2.5 数据库查询权限构建

**文件位置**：`packages/lib/utils/teams.ts:125-169`

```typescript
export const buildTeamWhereQuery = ({
  teamId,
  userId,
  roles,
}: BuildTeamWhereQueryOptions): Prisma.TeamWhereUniqueInput => {
  if (!roles) {
    return {
      id: teamId,
      teamGroups: {
        some: {
          organisationGroup: {
            organisationGroupMembers: {
              some: { organisationMember: { userId } },
            },
          },
        },
      },
    };
  }

  return {
    id: teamId,
    teamGroups: {
      some: {
        organisationGroup: {
          organisationGroupMembers: {
            some: { organisationMember: { userId } },
          },
        },
        teamRole: { in: roles },
      },
    },
  };
};
```

**作用**：构建Prisma查询条件，确保用户只能访问有权限的团队数据。

---

## 三、服务端权限获取链路

### 3.1 权限检查调用链

**获取团队信息时的完整权限检查链路：

```
1. Route Loader (apps/remix/app/routes/_authenticated+/t.$teamUrl+/settings._layout.tsx:21-31
   ↓
2. getTeamByUrl (packages/lib/server-only/team/get-team.ts:28-33
   ↓
3. getTeam (packages/lib/server-only/team/get-team.ts:38-84
   ↓
4. buildTeamWhereQuery (packages/lib/utils/teams.ts:125-169) → 数据库级权限过滤
   ↓
5. prisma.team.findFirst → 只返回用户有权访问的团队
   ↓
6. getHighestTeamRoleInGroup (packages/lib/utils/teams.ts:76-89) → 计算用户最高角色
   ↓
7. 返回 team.currentTeamRole
```

### 3.2 获取团队信息（附带角色

**文件位置**：`packages/lib/server-only/team/get-team.ts:38-84`

```typescript
export const getTeam = async ({ teamReference, userId }) => {
  const team = await prisma.team.findFirst({
    where: {
      ...buildTeamWhereQuery({ teamId: undefined, userId }),
      id: typeof teamReference === 'number' ? teamReference : undefined,
      url: typeof teamReference === 'string' ? teamReference : undefined,
    },
    include: {
      teamGroups: {
        where: {
          organisationGroup: {
            organisationGroupMembers: {
              some: { organisationMember: { userId } },
            },
          },
        },
      },
    },
  });

  return {
    ...team,
    currentTeamRole: getHighestTeamRoleInGroup(team.teamGroups),
  };
};
```

**关键点**：
- 通过关联查询`teamGroups`时加入用户过滤条件，确保只查询到的是该用户所属的用户组
- 通过`getHighestTeamRoleInGroup`计算用户在该团队的最高角色

### 3.3 文档查询中的权限过滤

**文件位置**：`packages/lib/server-only/document/find-documents.ts:299-336`

```typescript
const applyTeamFilters = (qb, teamData) => {
  const allowedVisibilities = match(teamData.currentTeamRole)
    .with(TeamMemberRole.ADMIN, () => [
      DocumentVisibility.EVERYONE,
      DocumentVisibility.MANAGER_AND_ABOVE,
      DocumentVisibility.ADMIN,
    ])
    .with(TeamMemberRole.MANAGER, () => [DocumentVisibility.EVERYONE, DocumentVisibility.MANAGER_AND_ABOVE])
    .otherwise(() => [DocumentVisibility.EVERYONE]);

  const visibilityFilter = (eb) =>
    eb.or([
      eb('Envelope.visibility', 'in', allowedVisibilities.map((v) => sql.lit(v))),
      eb('Envelope.userId', '=', user.id),
      recipientExists(eb, user.email),
    ]);

  // ... 其他过滤条件
};
```

**权限逻辑**：
1. 根据团队角色确定可见的文档范围
2. OR条件：满足文档可见性匹配 OR 用户是文档创建者 OR 用户是文档接收者

---

## 四、UI层权限收敛点

### 4.1 TeamSession类型定义

**文件位置**：`packages/trpc/server/organisation-router/get-organisation-session.types.ts:31`

```typescript
export type TeamSession = {
  id: number;
  name: string;
  url: string;
  currentTeamRole: TeamMemberRole;
  // ...其他字段
};
```

### 4.2 TeamProvider上下文注入

**文件位置**：`apps/remix/app/providers/team.tsx`

```typescript
const TeamContext = createContext<TeamSession | null>(null);

export const useCurrentTeam = () => {
  const context = useContext(TeamContext);
  if (!context) throw new Error('useCurrentTeam must be used within a TeamProvider');
  return context;
};

export const TeamProvider = ({ children, team }) => {
  return <TeamContext.Provider value={team}>{children}</TeamContext.Provider>;
};
```

**收敛作用**：将`team.currentTeamRole`注入整个团队上下文，所有子组件可直接获取。

### 4.3 路由级别的权限重定向

**文件位置**：`apps/remix/app/routes/_authenticated+/t.$teamUrl+/settings._layout.tsx:21-31`

```typescript
export async function loader({ request, params }) {
  const session = await getSession(request);
  const team = await getTeamByUrl({ userId: session.user.id, teamUrl: params.teamUrl });

  // 权限检查：无权限则重定向到团队首页
  if (!team || !canExecuteTeamAction('MANAGE_TEAM', team.currentTeamRole)) {
    throw redirect(`/t/${params.teamUrl}`);
  }
}
```

**收敛作用**：在服务端渲染前进行权限检查，阻止无权限用户访问设置页面。

### 4.4 组件级别的按钮禁用

**文件位置**：`apps/remix/app/components/tables/team-members-table.tsx:137-178`

```typescript
<TeamMemberUpdateDialog
  currentUserTeamRole={team.currentTeamRole}
  trigger={
    <DropdownMenuItem
      disabled={
        organisation.ownerUserId === row.original.userId ||
        !isTeamRoleWithinUserHierarchy(team.currentTeamRole, row.original.teamRole)
      }
    >
      <EditIcon className="mr-2 h-4 w-4" />
      <Trans>Update role</Trans>
    </DropdownMenuItem>
  }
/>
```

**收敛逻辑**：
1. 组织所有者不能被修改
2. 只能修改层级低于等于当前用户角色的成员

### 4.5 对话框级别的角色过滤

**文件位置**：`apps/remix/app/components/dialogs/team-member-update-dialog.tsx:94-110, 152-157`

```typescript
useEffect(() => {
  if (!open) return;
  form.reset();

  // 打开对话框时再次检查权限
  if (!isTeamRoleWithinUserHierarchy(currentUserTeamRole, memberTeamRole)) {
    setOpen(false);
    toast({ title: 'You cannot modify a team member who has a higher role than you.' });
  }
}, [open, currentUserTeamRole, memberTeamRole]);

// 下拉选择只显示层级内允许的角色
<SelectContent>
  {TEAM_MEMBER_ROLE_HIERARCHY[currentUserTeamRole].map((role) => (
    <SelectItem key={role} value={role}>
      {_(EXTENDED_TEAM_MEMBER_ROLE_MAP[role]) ?? role}
    </SelectItem>
  ))}
</SelectContent>
```

**收敛作用**：
1. 对话框打开时进行二次权限验证
2. 下拉选项根据当前用户角色层级进行过滤

---

## 五、角色、操作、可见性、UI入口对照表

| 角色 | 可执行操作 | 可见文档范围 | UI入口 |
|-----|----------|------------|-------|
| **ADMIN** | MANAGE_TEAM, DELETE_TEAM | EVERYONE, MANAGER_AND_ABOVE, ADMIN | 团队设置全部入口可见，成员管理全部功能完整，删除团队按钮可见 |
| **MANAGER** | MANAGE_TEAM | EVERYONE, MANAGER_AND_ABOVE | 团队设置入口可见，可管理成员但不能删除团队 |
| **MEMBER** | - | EVERYONE | 团队设置入口不可见，仅可见团队公开文档 |
| **所有者** | (组织级最高权限) | 全部可见 | 组织管理控制台可见 |
| **SIGNER** (文档级) | 签署文档 | 仅本人作为接收者的文档 | 文档签署页面 |
| **VIEWER** (文档级) | 查看文档 | 仅本人作为接收者的文档 | 文档查看页面 |
| **APPROVER** (文档级) | 审批文档 | 仅本人作为接收者的文档 | 文档审批页面 |

---

## 六、从配置到UI的最短调用链

### 6.1 团队设置页面权限检查最短链路

```
配置定义
  ↓
[packages/lib/constants/teams.ts]
  TEAM_MEMBER_ROLE_PERMISSIONS_MAP.MANAGE_TEAM
  ↓
工具函数
  ↓
[packages/lib/utils/teams.ts]
  canExecuteTeamAction('MANAGE_TEAM', role)
  ↓
服务端获取
  ↓
[packages/lib/server-only/team/get-team.ts]
  getTeamByUrl() → 返回 team.currentTeamRole
  ↓
路由Loader
  ↓
[apps/remix/app/routes/_authenticated+/t.$teamUrl+/settings._layout.tsx]
  loader() → canExecuteTeamAction 检查，无权限则 redirect
  ↓
UI组件
  ↓
[apps/remix/app/routes/_authenticated+/t.$teamUrl+/settings._layout.tsx]
  TeamsSettingsLayout → useCurrentTeam() → canExecuteTeamAction 二次检查
  ↓
子组件渲染
  ↓
设置页面完整渲染（仅对ADMIN/MANAGER可见）
```

### 6.2 成员更新权限检查最短链路

```
配置定义
  ↓
[packages/lib/constants/teams.ts]
  TEAM_MEMBER_ROLE_HIERARCHY
  ↓
工具函数
  ↓
[packages/lib/utils/teams.ts]
  isTeamRoleWithinUserHierarchy(currentRole, targetRole)
  ↓
表格行渲染
  ↓
[apps/remix/app/components/tables/team-members-table.tsx]
  DropdownMenuItem disabled=!isTeamRoleWithinUserHierarchy(...)
  ↓
对话框打开
  ↓
[apps/remix/app/components/dialogs/team-member-update-dialog.tsx]
  useEffect → isTeamRoleWithinUserHierarchy 二次检查
  ↓
角色选择下拉
  ↓
TEAM_MEMBER_ROLE_HIERARCHY[currentUserTeamRole].map()
  ↓
TRPC mutation
  ↓
[packages/trpc/server/team-router/update-team-member.ts]
  buildTeamWhereQuery 带 MANAGE_TEAM 权限角色
  isTeamRoleWithinUserHierarchy 服务端最终验证
```

---

## 七、中间件的真实实现

**文件位置**：`apps/remix/server/middleware.ts`

**重要说明**：当前`appMiddleware`仅处理以下逻辑，**不负责团队权限检查：

```typescript
export const appMiddleware = async (c: Context, next: Next) => {
  const { req } = c;
  const { path } = req;

  // 1. 处理重定向
  const redirectPath = await handleRedirects(c);
  if (redirectPath) return c.redirect(redirectPath);

  await next();

  // 2. 设置团队URL cookie
  if (pathname.startsWith('/t/')) {
    setCookie(c, 'preferred-team-url', pathname.split('/')[2], { sameSite: 'lax' });
  }
};
```

**权限检查位置**：权限检查分散在各个Route Loader和TRPC Procedure中，而非集中在中间件。
