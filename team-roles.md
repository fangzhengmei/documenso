# 团队角色权限系统设计

## 一、角色定义

### 1.1 系统角色类型

| 概念角色 | 系统中对应类型 | 定义文件 | 说明 |
|---------|-------------|---------|------|
| 所有者 | `Organisation.ownerUserId` | `packages/prisma/schema.prisma:719` | 组织拥有者，独立于团队角色体系 |
| 管理员 | `TeamMemberRole.ADMIN` | `packages/prisma/schema.prisma:822` | 团队管理员角色 |
| 经理 | `TeamMemberRole.MANAGER` | `packages/prisma/schema.prisma:823` | 团队经理角色 |
| 成员 | `TeamMemberRole.MEMBER` | `packages/prisma/schema.prisma:824` | 团队普通成员角色 |
| 文档参与者 | `RecipientRole.SIGNER/APPROVER` | `packages/prisma/schema.prisma:574` | 文档级接收者角色，非团队角色 |
| 查看人 | `RecipientRole.VIEWER` | `packages/prisma/schema.prisma:576` | 文档级只读角色，非团队角色 |

### 1.2 角色权限映射配置

**文件位置**: `packages/lib/constants/teams.ts:31-34`

```typescript
export const TEAM_MEMBER_ROLE_PERMISSIONS_MAP = {
  DELETE_TEAM: [TeamMemberRole.ADMIN],
  MANAGE_TEAM: [TeamMemberRole.ADMIN, TeamMemberRole.MANAGER],
} satisfies Record<string, TeamMemberRole[]>;
```

| 操作权限 | 允许角色 |
|---------|---------|
| DELETE_TEAM | ADMIN |
| MANAGE_TEAM | ADMIN, MANAGER |

### 1.3 角色层级关系

**文件位置**: `packages/lib/constants/teams.ts:48-52`

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

**文件位置**: `packages/lib/constants/teams.ts:36-40`

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

## 二、权限断言

### 2.1 操作权限检查函数

**文件位置**: `packages/lib/utils/teams.ts:43-48`

```typescript
export const canExecuteTeamAction = (
  action: keyof typeof TEAM_MEMBER_ROLE_PERMISSIONS_MAP,
  role: keyof typeof TEAM_MEMBER_ROLE_MAP,
) => {
  return TEAM_MEMBER_ROLE_PERMISSIONS_MAP[action].some((i) => i === role);
};
```

### 2.2 角色层级权限检查函数

**文件位置**: `packages/lib/utils/teams.ts:69-74`

```typescript
export const isTeamRoleWithinUserHierarchy = (
  currentUserRole: keyof typeof TEAM_MEMBER_ROLE_MAP,
  roleToCheck: keyof typeof TEAM_MEMBER_ROLE_MAP,
) => {
  return TEAM_MEMBER_ROLE_HIERARCHY[currentUserRole].some((i) => i === roleToCheck);
};
```

### 2.3 获取用户最高角色函数

**文件位置**: `packages/lib/utils/teams.ts:76-89`

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

### 2.4 数据库查询权限构建函数

**文件位置**: `packages/lib/utils/teams.ts:125-187`

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

---

## 三、服务端链路

### 3.1 获取团队信息链路

**文件位置**: `packages/lib/server-only/team/get-team.ts:38-84`

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

### 3.2 路由 Loader 权限检查

**文件位置**: `apps/remix/app/routes/_authenticated+/t.$teamUrl+/settings._layout.tsx:21-32`

```typescript
export async function loader({ request, params }) {
  const session = await getSession(request);

  const team = await getTeamByUrl({
    userId: session.user.id,
    teamUrl: params.teamUrl,
  });

  if (!team || !canExecuteTeamAction('MANAGE_TEAM', team.currentTeamRole)) {
    throw redirect(`/t/${params.teamUrl}`);
  }
}
```

### 3.3 TRPC Mutation 权限检查

**文件位置**: `packages/trpc/server/team-router/update-team-member.ts:28-141`

```typescript
const team = await prisma.team.findFirst({
  where: {
    AND: [
      buildTeamWhereQuery({
        teamId,
        userId,
        roles: TEAM_MEMBER_ROLE_PERMISSIONS_MAP['MANAGE_TEAM'],
      }),
    ],
  },
});

const { teamRole: currentUserTeamRole } = await getMemberRoles({
  teamId,
  reference: { type: 'User', id: userId },
});

const { teamRole: currentMemberToUpdateTeamRole } = await getMemberRoles({
  teamId,
  reference: { type: 'Member', id: memberId },
});

if (!isTeamRoleWithinUserHierarchy(currentUserTeamRole, currentMemberToUpdateTeamRole)) {
  throw new AppError(AppErrorCode.UNAUTHORIZED, {
    message: 'Cannot update a member with a higher role',
  });
}

if (!isTeamRoleWithinUserHierarchy(currentUserTeamRole, data.role)) {
  throw new AppError(AppErrorCode.UNAUTHORIZED, {
    message: 'Cannot update a member to a role higher than your own',
  });
}
```

### 3.4 中间件说明

**文件位置**: `apps/remix/server/middleware.ts:18-58`

**重要说明**: 当前 `appMiddleware` 不处理团队权限检查，仅处理重定向和设置团队URL cookie。权限检查分散在各个 Route Loader 和 TRPC Procedure 中。

---

## 四、UI收敛

### 4.1 TeamProvider 上下文注入

**文件位置**: `apps/remix/app/providers/team.tsx`

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

### 4.2 路由组件权限检查

**文件位置**: `apps/remix/app/routes/_authenticated+/t.$teamUrl+/settings._layout.tsx:97-118`

```typescript
if (!canExecuteTeamAction('MANAGE_TEAM', team.currentTeamRole)) {
  return (
    <GenericErrorLayout
      errorCode={401}
      errorCodeMap={{
        401: {
          heading: msg`Unauthorized`,
          subHeading: msg`401 Unauthorized`,
          message: msg`You are not authorized to access this page.`,
        },
      }}
    />
  );
}
```

### 4.3 组件按钮禁用控制

**文件位置**: `apps/remix/app/components/tables/team-members-table.tsx:137-178`

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

### 4.4 对话框角色选择过滤

**文件位置**: `apps/remix/app/components/dialogs/team-member-update-dialog.tsx:94-110, 152-157`

```typescript
useEffect(() => {
  if (!open) return;
  form.reset();

  if (!isTeamRoleWithinUserHierarchy(currentUserTeamRole, memberTeamRole)) {
    setOpen(false);
    toast({ title: 'You cannot modify a team member who has a higher role than you.' });
  }
}, [open, currentUserTeamRole, memberTeamRole]);

<SelectContent>
  {TEAM_MEMBER_ROLE_HIERARCHY[currentUserTeamRole].map((role) => (
    <SelectItem key={role} value={role}>
      {_(EXTENDED_TEAM_MEMBER_ROLE_MAP[role]) ?? role}
    </SelectItem>
  ))}
</SelectContent>
```

---

## 五、角色-操作-可见范围-UI入口对照表

| 角色 | 可执行操作 | 可见文档范围 | UI入口 |
|-----|----------|------------|-------|
| ADMIN | MANAGE_TEAM, DELETE_TEAM | EVERYONE, MANAGER_AND_ABOVE, ADMIN | 团队设置入口可见，成员管理完整功能，删除团队按钮可用 |
| MANAGER | MANAGE_TEAM | EVERYONE, MANAGER_AND_ABOVE | 团队设置入口可见，可管理成员但不可删除团队 |
| MEMBER | - | EVERYONE | 团队设置入口不可见，仅可见团队公开文档 |
| 所有者 | 组织级最高权限 | 全部可见 | 组织管理控制台可见 |
| SIGNER (文档级) | 签署文档 | 仅本人作为接收者的文档 | 文档签署页面 |
| VIEWER (文档级) | 查看文档 | 仅本人作为接收者的文档 | 文档查看页面 |
| APPROVER (文档级) | 审批文档 | 仅本人作为接收者的文档 | 文档审批页面 |

---

## 六、配置到UI的最短调用链

```
配置定义 (packages/lib/constants/teams.ts)
  ↓
TEAM_MEMBER_ROLE_PERMISSIONS_MAP.MANAGE_TEAM
  ↓
权限断言 (packages/lib/utils/teams.ts)
  ↓
canExecuteTeamAction('MANAGE_TEAM', role)
  ↓
服务端获取 (packages/lib/server-only/team/get-team.ts)
  ↓
getTeamByUrl() → 返回 team.currentTeamRole
  ↓
路由Loader检查 (apps/remix/app/routes/_authenticated+/t.$teamUrl+/settings._layout.tsx)
  ↓
loader() → canExecuteTeamAction 检查，无权限则 redirect
  ↓
UI组件二次检查 (同一文件)
  ↓
TeamsSettingsLayout → useCurrentTeam() → canExecuteTeamAction 二次检查
  ↓
页面渲染完成（仅对ADMIN/MANAGER可见）
```
