# 团队角色权限系统设计报告

## 一、角色定义

### 1.1 角色类型定义

系统中定义了三种团队角色，位于 `packages/lib/constants/teams.ts`：

```typescript
export const TEAM_MEMBER_ROLE_MAP: Record<keyof typeof TeamMemberRole, MessageDescriptor> = {
  ADMIN: msg`Admin`,         // 管理员
  MANAGER: msg`Manager`,     // 经理
  MEMBER: msg`Member`,         // 成员
};
```

### 1.2 角色权限映射

每种角色与操作权限映射定义了每个角色可以执行哪些操作：

```typescript
// packages/lib/constants/teams.ts
export const TEAM_MEMBER_ROLE_PERMISSIONS_MAP = {
  DELETE_TEAM: [TeamMemberRole.ADMIN],
  MANAGE_TEAM: [TeamMemberRole.ADMIN, TeamMemberRole.MANAGER],
} satisfies Record<string, TeamMemberRole[]>;
```

### 1.3 角色层级关系

角色层级定义了哪个角色可以管理其他角色：

```typescript
// packages/lib/constants/teams.ts
export const TEAM_MEMBER_ROLE_HIERARCHY = {
  [TeamMemberRole.ADMIN]: [TeamMemberRole.ADMIN, TeamMemberRole.MANAGER, TeamMemberRole.MEMBER],
  [TeamMemberRole.MANAGER]: [TeamMemberRole.MANAGER, TeamMemberRole.MEMBER],
  [TeamMemberRole.MEMBER]: [TeamMemberRole.MEMBER],
} satisfies Record<TeamMemberRole, TeamMemberRole[]>;
```

### 1.4 文档可见性权限

```typescript
export const TEAM_DOCUMENT_VISIBILITY_MAP = {
  [TeamMemberRole.ADMIN]: [DocumentVisibility.ADMIN, DocumentVisibility.MANAGER_AND_ABOVE, DocumentVisibility.EVERYONE],
  [TeamMemberRole.MANAGER]: [DocumentVisibility.MANAGER_AND_ABOVE, DocumentVisibility.EVERYONE],
  [TeamMemberRole.MEMBER]: [DocumentVisibility.EVERYONE],
} satisfies Record<TeamMemberRole, DocumentVisibility[]>;
```

## 二、权限断言工具函数

### 2.1 核心工具函数

位于 `packages/lib/utils/teams.ts`

#### `canExecuteTeamAction` - 操作权限检查

```typescript
export const canExecuteTeamAction = (
  action: keyof typeof TEAM_MEMBER_ROLE_PERMISSIONS_MAP,
  role: keyof typeof TEAM_MEMBER_ROLE_MAP,
) => {
  return TEAM_MEMBER_ROLE_PERMISSIONS_MAP[action].some((i) => i === role);
};
```

**使用场景**：检查用户是否可以执行特定操作，如 `MANAGE_TEAM`、`DELETE_TEAM`

#### `isTeamRoleWithinUserHierarchy` - 层级权限检查

```typescript
export const isTeamRoleWithinUserHierarchy = (
  currentUserRole: keyof typeof TEAM_MEMBER_ROLE_MAP,
  roleToCheck: keyof typeof TEAM_MEMBER_ROLE_MAP,
) => {
  return TEAM_MEMBER_ROLE_HIERARCHY[currentUserRole].some((i) => i === roleToCheck);
};
```

**使用场景**：检查当前用户是否可以修改另一个角色的用户（例如，Manager不能修改Admin）

#### `canAccessTeamDocument` - 文档可见性检查

```typescript
export const canAccessTeamDocument = (role: TeamMemberRole, visibility: DocumentVisibility) => {
  return TEAM_DOCUMENT_VISIBILITY_MAP[role].some((i) => i === visibility);
};
```

#### `getHighestTeamRoleInGroup` - 获取用户最高角色

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

### 2.2 数据库查询构建器

#### `buildTeamWhereQuery` - 构建带权限的查询条件

```typescript
export type BuildTeamWhereQueryOptions = {
  teamId: number | undefined;
  userId: number;
  roles?: TeamMemberRole[];
};

export const buildTeamWhereQuery = ({
  teamId,
  userId,
  roles,
}: BuildTeamWhereQueryOptions): Prisma.TeamWhereUniqueInput => {
  // 构建关联查询条件，确保用户只能访问有权限的团队
};
```

## 三、服务端权限获取与中间件

### 3.1 获取团队成员角色

位于 `packages/lib/server-only/team/get-member-roles.ts`

```typescript
export const getMemberRoles = async ({ teamId, reference }: GetMemberRolesOptions) => {
  const team = await prisma.team.findFirst({
    where: {
      id: teamId,
    },
    include: {
      teamGroups: {
        where: {
          organisationGroup: {
            organisationGroupMembers: {
              some: {
                organisationMember:
                  reference.type === 'User'
                    ? { userId: reference.id }
                    : { id: reference.id },
              },
            },
          },
        },
      },
    },
  });

  return {
    teamRole: getHighestTeamRoleInGroup(team.teamGroups),
  };
};
```

### 3.2 获取团队信息（附带当前用户角色

位于 `packages/lib/server-only/team/get-team.ts`

```typescript
export const getTeam = async ({ teamReference, userId }) => {
  const team = await prisma.team.findFirst({
    where: {
      ...buildTeamWhereQuery({ teamId: undefined, userId }),
      // ...
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
      // ...
    },
  });

  return {
    ...team,
    currentTeamRole: getHighestTeamRoleInGroup(team.teamGroups),
    // ...
  };
};
```

### 3.3 TRPC路由中的权限检查

以 `update-team-member.ts` 为例：

```typescript
export const updateTeamMemberRoute = authenticatedProcedure
  .input(ZUpdateTeamMemberRequestSchema)
  .mutation(async ({ ctx, input }) => {
    const { teamId, memberId, data } = input;
    const userId = ctx.user.id;

    // 1. 构建带权限的查询，只允许有权限的用户操作
    const team = await prisma.team.findFirst({
      where: {
        AND: [
          buildTeamWhereQuery({
            teamId,
            userId,
            roles: TEAM_MEMBER_ROLE_PERMISSIONS_MAP['MANAGE_TEAM'],
          }),
          // ...
        ],
      },
    });

    // 2. 获取当前用户和目标用户的角色
    const { teamRole: currentUserTeamRole } = await getMemberRoles({
      teamId,
      reference: { type: 'User', id: userId },
    });

    const { teamRole: currentMemberToUpdateTeamRole } = await getMemberRoles({
      teamId,
      reference: { type: 'Member', id: memberId },
    });

    // 3. 检查角色层级权限
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

    // 执行更新操作...
  });
```

## 四、UI层权限收敛

### 4.1 TeamProvider - 团队上下文提供

位于 `apps/remix/app/providers/team.tsx`

```typescript
type TeamProviderValue = TeamSession;

interface TeamProviderProps {
  children: React.ReactNode;
  team: TeamProviderValue | null;
}

const TeamContext = createContext<TeamProviderValue | null>(null);

export const useCurrentTeam = () => {
  const context = useContext(TeamContext);

  if (!context) {
    throw new Error('useCurrentTeam must be used within a TeamProvider');
  }

  return context;
};

export const TeamProvider = ({ children, team }: TeamProviderProps) => {
  return <TeamContext.Provider value={team}>{children}</TeamContext.Provider>;
};
```

### 4.2 路由级别的权限控制

位于 `apps/remix/app/routes/_authenticated+/t.$teamUrl+/settings._layout.tsx`

```typescript
// Server Loader 中的权限检查
export async function loader({ request, params }: Route.LoaderArgs) {
  const session = await getSession(request);

  const team = await getTeamByUrl({
    userId: session.user.id,
    teamUrl: params.teamUrl,
  });

  // 权限检查：无权限则重定向
  if (!team || !canExecuteTeamAction('MANAGE_TEAM', team.currentTeamRole)) {
    throw redirect(`/t/${params.teamUrl}`);
  }
}

// Client 组件中的权限检查
export default function TeamsSettingsLayout() {
  const team = useCurrentTeam();

  // 双重检查：客户端再次验证权限
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
        // ...
      />
    );
  }

  // 渲染设置页面...
}
```

### 4.3 组件级别的权限控制 - 团队成员表格

位于 `apps/remix/app/components/tables/team-members-table.tsx`

```typescript
export const TeamMembersTable = () => {
  const organisation = useCurrentOrganisation();
  const team = useCurrentTeam();

  const columns = useMemo(() => {
    return [
      // ... 其他列
      {
        header: _(msg`Actions`),
        cell: ({ row }) => (
          <DropdownMenu>
            <DropdownMenuTrigger>
              <MoreHorizontal className="h-5 w-5 text-muted-foreground" />
            </DropdownMenuTrigger>

            <DropdownMenuContent>
              {/* 更新角色按钮 - 根据层级权限禁用 */}
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

              {/* 删除成员按钮 - 根据层级权限禁用 */}
              <TeamMemberDeleteDialog
                trigger={
                  <DropdownMenuItem
                    disabled={
                      organisation.ownerUserId === row.original.userId ||
                      !isTeamRoleWithinUserHierarchy(team.currentTeamRole, row.original.teamRole)
                    }
                  >
                    <Trash2Icon className="mr-2 h-4 w-4" />
                    <Trans>Remove</Trans>
                  </DropdownMenuItem>
                }
              />
            </DropdownMenuContent>
          </DropdownMenu>
        ),
      },
    ];
  }, [groups]);

  // ...
};
```

### 4.4 对话框级别的权限控制

位于 `apps/remix/app/components/dialogs/team-member-update-dialog.tsx`

```typescript
export const TeamMemberUpdateDialog = ({
  currentUserTeamRole,
  memberTeamRole,
  // ...
}: TeamMemberUpdateDialogProps) => {
  const [open, setOpen] = useState(false);

  // 打开时再次检查权限
  useEffect(() => {
    if (!open) {
      return;
    }

    if (!isTeamRoleWithinUserHierarchy(currentUserTeamRole, memberTeamRole)) {
      setOpen(false);
      toast({
        title: _(msg`You cannot modify a team member who has a higher role than you.`),
        variant: 'destructive',
      });
    }
  }, [open, currentUserTeamRole, memberTeamRole]);

  // 下拉选择只显示层级内允许的角色
  return (
    <Dialog>
      <Form {...form}>
        <form onSubmit={form.handleSubmit(onFormSubmit)}>
          <FormField
            control={form.control}
            name="role"
            render={({ field }) => (
              <FormItem>
                <FormLabel required>
                  <Trans>Role</Trans>
                </FormLabel>
                <FormControl>
                  <Select {...field} onValueChange={field.onChange}>
                    <SelectTrigger>
                      <SelectValue />
                    </SelectTrigger>
                    <SelectContent>
                      {TEAM_MEMBER_ROLE_HIERARCHY[currentUserTeamRole].map((role) => (
                      <SelectItem key={role} value={role}>
                        {_(EXTENDED_TEAM_MEMBER_ROLE_MAP[role]) ?? role}
                      </SelectItem>
                    ))}
                  </SelectContent>
                </Select>
              </FormControl>
            </FormItem>
          )}
        />
        {/* ... */}
      </form>
    </Dialog>
  );
};
```

## 五、权限系统架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         配置层 (Configuration)                        │
├─────────────────────────────────────────────────────────────────┤
│  packages/lib/constants/teams.ts                              │
│  - TEAM_MEMBER_ROLE_PERMISSIONS_MAP                          │
│  - TEAM_MEMBER_ROLE_HIERARCHY                                  │
│  - TEAM_DOCUMENT_VISIBILITY_MAP                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                       工具函数层 (Utils)                       │
├─────────────────────────────────────────────────────────────────┤
│  packages/lib/utils/teams.ts                                 │
│  - canExecuteTeamAction(action, role)                         │
│  - isTeamRoleWithinUserHierarchy(currentRole, targetRole)      │
│  - canAccessTeamDocument(role, visibility)                    │
│  - getHighestTeamRoleInGroup(groups)                           │
│  - buildTeamWhereQuery(options)                               │
└─────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
┌─────────────────────────────┐   ┌─────────────────────────────┐
│     服务端获取层         │   │     中间件/路由层         │
│   (Server Fetch)        │   │   (Middleware/Route)      │
├─────────────────────────────┤   ├─────────────────────────────┤
│ packages/lib/server-only/ │   │ apps/remix/server/       │
│ team/get-member-roles.ts  │   │ middleware.ts            │
│ team/get-team.ts         │   │ apps/remix/routes/      │
│ - 从数据库获取用户角色    │   │ - Loader中调用权限检查    │
│ - 关联查询团队组           │   │ - 无权限重定向           │
└─────────────────────────────┘   └─────────────────────────────┘
              │                               │
              └───────────────┬───────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     TRPC API 层                                │
├─────────────────────────────────────────────────────────────────┤
│  packages/trpc/server/team-router/*.ts                        │
│  - 使用 buildTeamWhereQuery 构建查询                           │
│  - 获取用户角色进行层级检查                                        │
│  - 抛出 AppErrorCode.UNAUTHORIZED                                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     UI 上下文层 (Context)                      │
├─────────────────────────────────────────────────────────────────┤
│  apps/remix/app/providers/team.tsx                          │
│  - TeamProvider 提供 team.currentTeamRole                   │
│  - useCurrentTeam() Hook                                    │
└─────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
┌─────────────────────────────┐   ┌─────────────────────────────┐
│     路由页面层             │   │     组件对话框层           │
│   (Route Pages)          │   │   (Components/Dialogs)    │
├─────────────────────────────┤   ├─────────────────────────────┤
│ apps/remix/routes/...    │   │ apps/remix/components/   │
│ - Loader 权限检查           │   │ - 按钮禁用状态        │
│ - 页面级渲染控制         │   │ - 菜单项显示/隐藏      │
│ - 错误页面展示           │   │ - 下拉选项过滤        │
└─────────────────────────────┘   └─────────────────────────────┘
```

## 六、核心设计原则

### 6.1 多层防护机制

1. **配置层**：统一的角色权限映射定义
2. **服务端Loader**：首次权限检查 + 无权限重定向
3. **TRPC API层**：数据库查询级别的权限过滤 + 业务逻辑权限检查
4. **UI组件层**：禁用/隐藏无权限操作按钮和菜单项

### 6.2 角色层级设计

- Admin > Manager > Member

- 高角色可以管理低角色，但低角色不能管理高角色
- 同级别之间不能互相管理

### 6.3 权限收敛点

1. **数据获取时收敛**：通过 `buildTeamWhereQuery` 在数据库查询层面过滤数据
2. **操作执行前收敛**：通过 `canExecuteTeamAction` 检查操作权限
3. **用户管理时收敛**：通过 `isTeamRoleWithinUserHierarchy` 检查层级权限
4. **UI渲染时收敛**：通过禁用/隐藏UI元素防止用户看到无权限操作

## 七、权限检查流程

```
用户访问团队设置页面
    │
    ▼
Route Loader 执行
    │
    ├─► getTeamByUrl() 获取团队信息
    │   └─► 构建带权限的数据库查询
    │
    └─► canExecuteTeamAction('MANAGE_TEAM', team.currentTeamRole)
        └─► 无权限 → 重定向到团队首页
        └─► 有权限 → 继续
    │
    ▼
Client Component 渲染
    │
    └─► useCurrentTeam() 获取团队上下文
    │
    └─► canExecuteTeamAction('MANAGE_TEAM', team.currentTeamRole)
        └─► 无权限 → 显示401错误页面
        └─► 有权限 → 渲染设置页面
    │
    ▼
子组件渲染（如成员表格）
    │
    └─► isTeamRoleWithinUserHierarchy(currentRole, targetRole)
        └─► 无权限 → 禁用更新/删除按钮
        └─► 有权限 → 启用按钮
    │
    ▼
用户点击更新角色
    │
    └─► TeamMemberUpdateDialog 打开
        └─► useEffect 再次检查层级权限
        └─► 下拉选项仅显示层级内允许的角色
    │
    ▼
提交更新请求
    │
    └─► TRPC updateTeamMemberRoute 执行
        ├─► buildTeamWhereQuery 带 MANAGE_TEAM 权限角色
        ├─► getMemberRoles 获取当前用户和目标用户角色
        ├─► isTeamRoleWithinUserHierarchy 两次检查
        └─► 执行更新操作
```