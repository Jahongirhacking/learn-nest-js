---
tags: [auth, level/intermediate]
level: intermediate
topic: Authorization
---

# RBAC

> **Bir jumlada:** RBAC (Role Based Access Control) — huquqni userga emas, **role**ga bog'lash; FotSpot esa undan bir qadam narida: role → permission.

**Before learning:** [[Authorization]], [[Database Relations]]
**After learning:** [[Roles and Permissions]], [[FotSpot Guard Chain]]

> 💡 **Tushunmadingmi? Express'da qo'lda qanday yozilishini ko'r:** [[Express Authorization Middleware]]


---

## 1. Definition

Oddiy model:

```text
User → role
```

RBAC + permission modeli:

```text
User
 └── UserRole
       └── Role
             └── RolePermission
                   └── Permission
```

## 2. Why — nega kerak?

Sodda `user.role = 'admin'` yondashuvi tez buziladi:

```text
"Adminlar video moderatsiya qilsin, lekin role boshqara olmasin"
"Ba'zi coachlar trial natijasini tahrirlay olsin"
"Yangi huquq qo'shildi — kodni qayta deploy qilmaslik kerak"
```

Permission qatlami bularni **kod o'zgarishisiz** hal qilish imkonini beradi: yangi `Permission` yozuvi qo'shiladi va kerakli `Role`ga biriktiriladi.

## 3. Internal mental model

```text
Role        = userning katta toifasi          (kim?)
Permission  = aniq action                     (nima qila oladi?)
```

Nega ikkalasi ham kerak:

```text
faqat Role bo'lsa       → har bir yangi huquq uchun kod o'zgaradi
faqat Permission bo'lsa → har bir userga o'nlab permission biriktirish kerak
ikkalasi birga          → role permissionlarni guruhlaydi
```

Ko'p-ko'pga (`many-to-many`) bog'lanish shuning uchun kerak: bir user bir necha rolga, bir role bir necha permissionga ega bo'lishi mumkin ([[Database Relations]]).

## 4. FotSpot implementation

Rollar:

```text
player · scout · coach · academy_manager · admin · super_admin
```

Permission misollari:

```text
admin:manage_roles
admin:moderate_video
admin:manage_users
trial:create
recommendation:review
```

Nomlash konvensiyasi `domain:action` — o'qish oson va guruhlash oson.

Guard zanjiri:

```text
JwtAuthGuard      → user kim
RolesGuard        → role mos keladimi
PermissionsGuard  → aniq permission bormi
```

## 5. Code example

Prisma schema (soddalashtirilgan):

```prisma
model User {
  id    String     @id @default(uuid())
  email String     @unique
  roles UserRole[]
}

model Role {
  id          String           @id @default(uuid())
  name        String           @unique
  users       UserRole[]
  permissions RolePermission[]
}

model Permission {
  id    String           @id @default(uuid())
  name  String           @unique      // "admin:moderate_video"
  roles RolePermission[]
}

model UserRole {
  userId String
  roleId String
  user   User @relation(fields: [userId], references: [id], onDelete: Cascade)
  role   Role @relation(fields: [roleId], references: [id], onDelete: Cascade)
  @@id([userId, roleId])
}

model RolePermission {
  roleId       String
  permissionId String
  role       Role       @relation(fields: [roleId], references: [id], onDelete: Cascade)
  permission Permission @relation(fields: [permissionId], references: [id], onDelete: Cascade)
  @@id([roleId, permissionId])
}
```

Userning permissionlarini yig'ish:

```ts
const user = await this.prisma.user.findUnique({
  where: { id },
  include: { roles: { include: { role: { include: { permissions: { include: { permission: true } } } } } } },
});

const permissions = new Set(
  user.roles.flatMap(ur => ur.role.permissions.map(rp => rp.permission.name)),
);
```

## 6. Common mistakes

- **Permissionni har bir requestda bir necha `include` bilan o'qish** → og'ir query. Tokenga solish yoki [[Redis Cache]] da saqlash kerak.
- **Permissionni faqat tokenga solib, hech qachon yangilamaslik** → huquq olib qo'yilsa ham token muddatigacha ishlayveradi. Yechim: qisqa TTL yoki cache invalidation.
- **Role nomini kodda string sifatida tarqatib yuborish** → enum/konstanta ishlat.
- **`super_admin` uchun alohida "hamma narsa mumkin" shartini unutish** → har bir permissionni qo'lda biriktirishga to'g'ri keladi.
- **RBAC bilan ownership'ni chalkashtirish** → "coach" bo'lish "**bu** trial'ni tahrirlash" degani emas ([[Authorization]]).

## 7. Practice

1. Yuqoridagi schema'ni yozib, migration qil ([[Prisma Migrations]]).
2. Seed script yoz: 6 ta role, 5 ta permission, bog'lanishlar.
3. Userning barcha permissionlarini qaytaruvchi service methodini yoz.
4. `PermissionsGuard` yozib, `admin:moderate_video` talab qiluvchi endpoint yasab ko'r.
5. Permissionlarni Redis'da 5 daqiqaga cache qil va invalidation'ni o'yla.

## 8. Related concepts

[[Roles and Permissions]] · [[Authorization]] · [[NestJS Guards]] · [[Database Relations]] · [[Prisma Schema]] · [[FotSpot Guard Chain]]
