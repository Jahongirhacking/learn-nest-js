---
tags: [auth, level/intermediate]
level: intermediate
topic: Authorization
---

# Roles and Permissions

> **Bir jumlada:** Role — userning **katta toifasi**, Permission — unga berilgan **aniq action**; ikkalasini ajratish keyinchalik kodni o'zgartirmasdan huquqlarni boshqarish imkonini beradi.

**Before learning:** [[RBAC]], [[Authorization]]
**After learning:** [[FotSpot Guard Chain]], [[NestJS Custom Decorators]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Authorization Middleware]]


---

## 1. Definition

```text
Role:        admin
Permissions: admin:manage_roles
             admin:moderate_video
             admin:manage_users
```

## 2. Why — nega kerak?

Savol: "Nega faqat role yetmaydi?"

Tasavvur qil: `admin` roli bor, lekin yangi talab keldi —

> "Moderator video tekshirsin, lekin userlarni o'chira olmasin."

**Faqat role bilan:** yangi `moderator` roli qo'shasan, keyin kodda har joyda `if (role === 'admin' || role === 'moderator')` yozasan. Har yangi talab — yangi deploy.

**Permission bilan:** `moderator` roli yaratiladi va unga faqat `admin:moderate_video` biriktiriladi. Kod o'zgarmaydi, chunki endpoint allaqachon shunday yozilgan:

```ts
@Permissions('admin:moderate_video')
```

## 3. Internal mental model

```text
Role       → "kim bu odam?"        → guruhlash mexanizmi
Permission → "nima qila oladi?"    → tekshiriladigan haqiqiy birlik
```

Qoida:

```text
Endpointlar PERMISSION bo'yicha himoyalansin.
Role — permissionlarni guruhlash uchun vosita, xolos.
```

Nomlash konvensiyasi:

```text
domain:action
  admin:manage_users
  trial:create
  media:verify
  recommendation:review
```

Yaxshi nom keyinchalik audit va debug'ni osonlashtiradi.

## 4. FotSpot implementation

```text
Rollar:
  player · scout · coach · academy_manager · admin · super_admin

Permission misollari:
  admin:manage_roles · admin:moderate_video · admin:manage_users
```

Guardlar ketma-ketligi shuni aks ettiradi:

```text
RolesGuard        → tez, arzon, keng filtr
PermissionsGuard  → aniq, nozik tekshiruv
```

Ya'ni `@Roles('admin')` "kirish darvozasi", `@Permissions('admin:moderate_video')` esa "xonaning kaliti".

## 5. Code example

```ts
export const ROLES_KEY = 'roles';
export const PERMISSIONS_KEY = 'permissions';

export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);
export const Permissions = (...perms: string[]) => SetMetadata(PERMISSIONS_KEY, perms);
```

```ts
@Controller('moderation')
export class ModerationController {
  @Roles('admin', 'super_admin')
  @Permissions('admin:moderate_video')
  @Patch('media/:id/verify')
  verify(@Param('id') id: string, @CurrentUser('id') adminId: string) {
    return this.moderation.verify(id, adminId);
  }

  @Roles('super_admin')
  @Permissions('admin:manage_roles')
  @Post('roles/:roleId/permissions/:permissionId')
  attach(@Param('roleId') roleId: string, @Param('permissionId') pid: string) {
    return this.rbac.attachPermission(roleId, pid);
  }
}
```

Permissionlarni doimiy string sifatida emas, konstanta orqali ishlat:

```ts
export const PERMISSION = {
  MODERATE_VIDEO: 'admin:moderate_video',
  MANAGE_ROLES: 'admin:manage_roles',
} as const;
```

## 6. Common mistakes

- **Permission nomlarini har joyda qo'lda yozish** → bitta harf xato = jimgina 403 yoki jimgina ochiq endpoint.
- **Role tekshiruvini kod ichida `if` bilan yozish** → guard/metadata naqshi buziladi, audit qilib bo'lmaydi.
- **Permissionni faqat frontendda yashirish** → tugma ko'rinmaydi, lekin endpoint ochiq qoladi.
- **Har bir user uchun individual permission berish** → RBAC'ning maqsadi yo'qoladi; role qatlami buning uchun bor.
- **`super_admin` uchun bypass qoidasini guard'ga qo'shmaslik yoki aksincha — hamma joyda bypass qilish.**

## 7. Practice

1. `@Roles()` va `@Permissions()` decoratorlarini konstantalar bilan yoz.
2. `moderator` degan yangi role yaratib, unga faqat `admin:moderate_video` ber.
3. O'sha user bilan `admin:manage_users` talab qiluvchi endpointga urin — 403 bo'lishi kerak.
4. Kodni **o'zgartirmasdan** yangi permission qo'shib ko'r: faqat DB yozuvi orqali.
5. `super_admin` uchun bypass qoidasini guard'ga qo'sh va xavfini muhokama qil.

## 8. Related concepts

[[RBAC]] · [[Authorization]] · [[NestJS Custom Decorators]] · [[NestJS Guards]] · [[FotSpot Guard Chain]] · [[Prisma Schema]]
