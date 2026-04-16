1. Package.json
json
{
  "name": "marketplace",
  "version": "1.0.0",
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "db:push": "prisma db push",
    "db:studio": "prisma studio"
  },
  "dependencies": {
    "@prisma/client": "^5.15.0",
    "@auth/prisma-adapter": "^1.2.0",
    "bcryptjs": "^2.4.3",
    "next": "^14.2.0",
    "next-auth": "^4.24.7",
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "stripe": "^16.2.0",
    "uploadthing": "^6.5.2"
  },
  "devDependencies": {
    "@types/bcryptjs": "^2.4.6",
    "@types/node": "^20.14.2",
    "@types/react": "^18.3.3",
    "@types/react-dom": "^18.3.0",
    "prisma": "^5.15.0",
    "typescript": "^5.5.3"
  }
}
2. Prisma Schema (prisma/schema.prisma)
text
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Account {
  id                String  @id @default(cuid())
  userId            String
  type              String
  provider          String
  providerAccountId String
  refresh_token     String? @db.Text
  access_token      String? @db.Text
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String? @db.Text
  session_state     String?

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([provider, providerAccountId])
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model User {
  id            String    @id @default(cuid())
  name          String?
  email         String    @unique
  emailVerified DateTime?
  image         String?
  role          Role      @default(BUYER)
  products      Product[]
  orders        Order[]
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt

  accounts Account[]
  sessions Session[]
}

model VerificationToken {
  identifier String
  token      String   @unique
  expires    DateTime

  @@unique([identifier, token])
}

enum Role {
  BUYER
  SELLER
  ADMIN
}

model Product {
  id          String   @id @default(cuid())
  title       String
  description String
  price       Int
  imageUrl    String?
  category    String?
  sellerId    String
  seller      User     @relation(fields: [sellerId], references: [id])
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
}

model Order {
  id          String   @id @default(cuid())
  amount      Int
  status      String   @default("PENDING")
  stripeId    String?
  buyerId     String
  buyer       User     @relation(fields: [buyerId], references: [id])
  createdAt   DateTime @default(now())
}
3. Environment (.env)
text
DATABASE_URL="postgresql://user:pass@localhost:5432/marketplace"
NEXTAUTH_SECRET="your-secret-key-here"
NEXTAUTH_URL="http://localhost:3000"
STRIPE_SECRET_KEY="sk_test_..."
STRIPE_WEBHOOK_SECRET="whsec_..."
4. Auth Config (lib/auth.ts)
ts
import NextAuth from "next-auth"
import CredentialsProvider from "next-auth/providers/credentials"
import { PrismaAdapter } from "@auth/prisma-adapter"
import { prisma } from "./prisma"
import bcrypt from "bcryptjs"

export const { handlers, signIn, signOut, auth } = NextAuth({
  adapter: PrismaAdapter(prisma),
  providers: [
    CredentialsProvider({
      credentials: {
        email: { label: "Email", type: "email" },
        password: { label: "Password", type: "password" }
      },
      async authorize(credentials) {
        if (!credentials?.email || !credentials?.password) return null

        const user = await prisma.user.findUnique({
          where: { email: credentials.email }
        })

        if (!user || !user.password) return null

        const isValid = await bcrypt.compare(credentials.password as string, user.password)
        if (!isValid) return null

        return user
      }
    })
  ],
  pages: {
    signIn: '/login'
  },
  session: { strategy: "jwt" }
})
5. Key Pages
Home (app/page.tsx)
tsx
'use client'
import { useSession } from "next-auth/react"
import Link from "next/link"

export default function Home() {
  const { data: session } = useSession()

  return (
    <main className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 p-8">
      <div className="max-w-6xl mx-auto">
        <h1 className="text-5xl font-bold text-gray-900 mb-8">Your Marketplace</h1>
        <p className="text-xl text-gray-600 mb-12">Buy and sell with trusted sellers</p>
        
        <div className="grid md:grid-cols-3 gap-6 mb-12">
          <Link href="/products" className="p-8 bg-white rounded-xl shadow-lg hover:shadow-xl transition-all">
            <h3 className="text-2xl font-semibold mb-2">🛒 Browse Products</h3>
            <p>Discover amazing products</p>
          </Link>
          <Link href="/sell" className="p-8 bg-white rounded-xl shadow-lg hover:shadow-xl transition-all">
            <h3 className="text-2xl font-semibold mb-2">💰 Sell Your Products</h3>
            <p>List items for sale</p>
          </Link>
          {session?.user.role !== 'BUYER' && (
            <Link href="/dashboard" className="p-8 bg-white rounded-xl shadow-lg hover:shadow-xl transition-all">
              <h3 className="text-2xl font-semibold mb-2">📊 Dashboard</h3>
              <p>Manage your store</p>
            </Link>
          )}
        </div>
        
        {!session && (
          <div className="text-center">
            <Link href="/login" className="bg-indigo-600 text-white px-8 py-4 rounded-xl text-xl font-semibold hover:bg-indigo-700">
              Sign up to start shopping
            </Link>
          </div>
        )}
      </div>
    </main>
  )
}
Products (app/products/page.tsx)
tsx
import { prisma } from "@/lib/prisma"
import Link from "next/link"

export default async function ProductsPage() {
  const products = await prisma.product.findMany({
    orderBy: { createdAt: "desc" },
    include: { seller: true }
  })

  return (
    <main className="min-h-screen p-8 max-w-7xl mx-auto">
      <div className="flex justify-between items-center mb-12">
        <h1 className="text-4xl font-bold">Products</h1>
        <Link href="/sell" className="bg-green-600 text-white px-6 py-3 rounded-xl hover:bg-green-700">
          + List Product
        </Link>
      </div>
      
      <div className="grid md:grid-cols-3 lg:grid-cols-4 gap-8">
        {products.map(product => (
          <div key={product.id} className="bg-white rounded-2xl shadow-lg hover:shadow-2xl transition-all overflow-hidden">
            <img src={product.imageUrl || '/placeholder.jpg'} alt={product.title} className="w-full h-64 object-cover" />
            <div className="p-6">
              <h3 className="font-bold text-xl mb-2 line-clamp-2">{product.title}</h3>
              <p className="text-gray-600 mb-4 line-clamp-2">{product.description}</p>
              <div className="flex justify-between items-center">
                <span className="text-2xl font-bold text-green-600">${product.price / 100}</span>
                <Link href={`/products/${product.id}`} className="bg-indigo-600 text-white px-4 py-2 rounded-xl text-sm hover:bg-indigo-700">
                  Buy Now
                </Link>
              </div>
              <p className="text-sm text-gray-500 mt-2">by {product.seller.name}</p>
            </div>
          </div>
        ))}
      </div>
    </main>
  )
}



