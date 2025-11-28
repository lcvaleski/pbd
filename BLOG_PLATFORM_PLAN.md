# Multi-Tenant Blog Platform Architecture Plan

## Current Architecture Analysis

### stoopside-cms (Admin/CMS)
- **Framework**: Next.js 14 with TypeScript
- **Auth**: NextAuth with Google OAuth (to be replaced)
- **Database**: Supabase (PostgreSQL)
- **Tables**: `site_settings` (title, subtitle) and `posts` (blog content)
- **API**: `/api/content` endpoint serves blog data with CORS enabled
- **Hosting**: Vercel

### stoopsidescribbles.com (Public Frontend)
- **Type**: Static HTML site hosted on GitHub Pages
- **Data Fetching**: JavaScript fetch() to CMS API
- **Newsletter**: Buttondown integration
- **Hosting**: GitHub Pages with custom domain

## New Architecture: Email/Password Auth with Live Preview Onboarding

### 1. Database Schema

```sql
-- Users table (replacing Google OAuth)
CREATE TABLE users (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  email TEXT UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,
  email_verified BOOLEAN DEFAULT false,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Tenants table for multi-tenant setup
CREATE TABLE tenants (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  owner_id UUID REFERENCES users(id),
  subdomain TEXT UNIQUE NOT NULL,
  custom_domain TEXT UNIQUE,
  blog_name TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  plan_tier TEXT DEFAULT 'free', -- free/pro
  storage_used BIGINT DEFAULT 0
);

-- Site settings with tenant association
CREATE TABLE site_settings (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  tenant_id UUID REFERENCES tenants(id),
  title TEXT NOT NULL,
  subtitle TEXT,
  theme_config JSONB,
  header_image_url TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Posts with tenant association
CREATE TABLE posts (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  tenant_id UUID REFERENCES tenants(id),
  title TEXT NOT NULL,
  content TEXT NOT NULL,
  date DATE NOT NULL DEFAULT CURRENT_DATE,
  published BOOLEAN DEFAULT false,
  is_pinned BOOLEAN DEFAULT false,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Temporary registration sessions
CREATE TABLE registration_sessions (
  session_id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  email TEXT,
  blog_name TEXT,
  subdomain TEXT,
  theme_choice TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  expires_at TIMESTAMPTZ DEFAULT NOW() + INTERVAL '24 hours'
);

-- Media uploads tracking
CREATE TABLE media_uploads (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  tenant_id UUID REFERENCES tenants(id),
  file_url TEXT NOT NULL,
  size_bytes BIGINT,
  uploaded_at TIMESTAMPTZ DEFAULT NOW()
);

-- Row Level Security Policies
CREATE POLICY "Users can only see own tenant data"
ON posts
FOR ALL
USING (tenant_id = current_setting('app.current_tenant')::uuid);

CREATE POLICY "Users can only modify own settings"
ON site_settings
FOR ALL
USING (tenant_id = current_setting('app.current_tenant')::uuid);
```

### 2. Sign-Up Flow with Live Blog Preview

#### Visual Layout
```
┌─────────────────────────────────────────┐
│         Sign-Up Form (30-40%)          │
│  ┌───────────────────────────────────┐  │
│  │  Create Your Blog                 │  │
│  │                                   │  │
│  │  Email: [___________]            │  │
│  │  Password: [___________]          │  │
│  │  Confirm: [___________]           │  │
│  │                                   │  │
│  │  Blog Name: [___________]         │  │
│  │  Subdomain: [_____].blog.com      │  │
│  │                                   │  │
│  │  [Create My Blog →]               │  │
│  └───────────────────────────────────┘  │
├─────────────────────────────────────────┤
│      Live Blog Preview (60-70%)         │
│  ┌───────────────────────────────────┐  │
│  │  [Your Blog Name Here]            │  │
│  │  Welcome to your blog!            │  │
│  │                                   │  │
│  │  📝 First Post Title              │  │
│  │  📝 Another Great Post            │  │
│  │  📝 Welcome to My Blog            │  │
│  │                                   │  │
│  │  Updates in real-time as          │  │
│  │  user types blog name ↑           │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

#### Implementation Details

1. **Real-Time Preview Updates**
```javascript
// Onboarding component
const [blogName, setBlogName] = useState("Your Blog");
const [subdomain, setSubdomain] = useState("");

// Send updates to preview iframe
useEffect(() => {
  const iframe = document.getElementById('blog-preview');
  iframe?.contentWindow?.postMessage({
    type: 'UPDATE_PREVIEW',
    blogName: blogName,
    subdomain: subdomain,
    theme: selectedTheme
  }, '*');
}, [blogName, subdomain, selectedTheme]);
```

2. **Preview Route** (`/preview/[sessionId]`)
```javascript
// Shows temporary blog with sample content
const samplePosts = [
  { title: "Welcome to My Blog!", date: "Today", preview: true },
  { title: "My First Adventure", date: "Yesterday", preview: true },
  { title: "Thoughts on Writing", date: "2 days ago", preview: true }
];

// Listen for updates from parent frame
window.addEventListener('message', (event) => {
  if (event.data.type === 'UPDATE_PREVIEW') {
    updateBlogTitle(event.data.blogName);
    updateTheme(event.data.theme);
  }
});
```

3. **Subdomain Availability Check**
```javascript
// Real-time validation
const checkSubdomain = async (subdomain) => {
  const res = await fetch(`/api/check-subdomain?s=${subdomain}`);
  const { available, suggestions } = await res.json();

  if (!available) {
    showSuggestions(suggestions); // "Try: coolblog123, coolblog-2024"
  }
  return available;
};
```

### 3. Authentication Flow

1. **Registration Process**
   - User fills form while seeing live preview
   - Password hashed with bcrypt
   - Account created but marked unverified
   - Verification email sent
   - Blog skeleton goes live immediately at subdomain
   - Full access granted after email verification

2. **NextAuth Configuration Update**
```javascript
// Replace Google Provider with Credentials
import CredentialsProvider from "next-auth/providers/credentials";
import bcrypt from "bcryptjs";

export default NextAuth({
  providers: [
    CredentialsProvider({
      name: "credentials",
      credentials: {
        email: { label: "Email", type: "email" },
        password: { label: "Password", type: "password" }
      },
      async authorize(credentials) {
        const user = await getUserByEmail(credentials.email);
        if (user && await bcrypt.compare(credentials.password, user.password_hash)) {
          return {
            id: user.id,
            email: user.email,
            tenantId: user.tenant_id
          };
        }
        return null;
      }
    })
  ],
  callbacks: {
    async session({ session, token }) {
      session.tenantId = token.tenantId;
      return session;
    },
    async jwt({ token, user }) {
      if (user) {
        token.tenantId = user.tenantId;
      }
      return token;
    }
  }
});
```

### 4. Multi-Tenant Routing

#### Dynamic Subdomain Handling
```javascript
// middleware.ts
export async function middleware(request: NextRequest) {
  const url = request.nextUrl;
  const hostname = request.headers.get('host');

  // Extract subdomain
  const subdomain = hostname?.split('.')[0];

  if (subdomain && subdomain !== 'www' && subdomain !== 'app') {
    // Rewrite to tenant-specific route
    url.pathname = `/site/${subdomain}${url.pathname}`;
    return NextResponse.rewrite(url);
  }
}
```

#### Vercel Configuration
```json
{
  "rewrites": [
    {
      "source": "/:path*",
      "has": [{"type": "host", "value": "(?<tenant>.*).yourdomain.com"}],
      "destination": "/api/site/:tenant/:path*"
    }
  ]
}
```

### 5. Directory Structure

```
/app
  /onboard                  # New user signup with preview
    /page.tsx              # Split screen signup
    /preview/[session]     # Blog preview iframe
  /auth                    # Authentication pages
    /signin
    /signup
    /forgot-password
    /verify-email
  /dashboard               # Authenticated CMS area
    /[tenantId]
      /posts
      /settings
      /media
  /site                    # Public blog routes
    /[subdomain]
      /page.tsx           # Blog homepage
      /post/[id]          # Individual posts
  /api
    /auth                 # Auth endpoints
      /register
      /login
      /reset-password
      /verify
    /tenant              # Tenant management
      /check-subdomain
      /create
    /[tenantId]          # Tenant-scoped APIs
      /content
      /posts
      /upload
```

### 6. Deployment Strategy

#### Option A: Hybrid ISR (Recommended)
- Use Incremental Static Regeneration
- Cache public pages, revalidate on changes
- Benefits: Fast, scalable, cost-effective

```javascript
// Blog page with ISR
export async function generateStaticParams() {
  const tenants = await getActiveTenants();
  return tenants.map(t => ({ subdomain: t.subdomain }));
}

export const revalidate = 60; // Revalidate every 60 seconds
```

#### Option B: Edge Functions
- Deploy tenant logic at edge
- Use Vercel Edge Functions or Cloudflare Workers
- Benefits: Low latency globally

#### Option C: Static Generation
- Build static sites per tenant
- Deploy to CDN
- Trigger rebuilds on content changes
- Benefits: Cheapest option for many inactive blogs

### 7. User Journey

1. **Discovery** → Landing page showcasing platform
2. **Sign Up** → See live preview while registering
3. **Customize** → Choose theme, add first post
4. **Launch** → Blog live at subdomain.yourdomain.com
5. **Grow** → Add content, customize further
6. **Upgrade** → Custom domain on pro plan

### 8. Key Features to Implement

#### Phase 1: MVP
- [x] Email/password authentication
- [x] Basic multi-tenant support
- [x] Live preview during signup
- [x] Simple blog theme
- [x] Post creation/editing
- [x] Subdomain routing

#### Phase 2: Enhancement
- [ ] Multiple themes
- [ ] Custom domain support
- [ ] Email notifications
- [ ] Analytics dashboard
- [ ] Media library
- [ ] SEO optimization

#### Phase 3: Growth
- [ ] Billing integration (Stripe)
- [ ] Advanced themes marketplace
- [ ] Plugin system
- [ ] API for external integrations
- [ ] Mobile app for content creation
- [ ] Collaborative editing

### 9. Security Considerations

1. **Tenant Isolation**
   - Row-level security in Supabase
   - Validate tenant context in all API routes
   - Separate media storage per tenant

2. **Authentication Security**
   - Bcrypt for password hashing
   - Rate limiting on login attempts
   - Email verification required
   - Password reset tokens expire
   - Session management with JWT

3. **Content Security**
   - XSS prevention with content sanitization
   - CSRF protection
   - File upload restrictions
   - CDN for static assets

### 10. Monitoring & Analytics

1. **Technical Metrics**
   - Response times per tenant
   - Database query performance
   - Storage usage per tenant
   - API rate limit tracking

2. **Business Metrics**
   - Sign-up conversion rate
   - Active blogs vs dormant
   - Content creation frequency
   - User retention

3. **Infrastructure**
   - Vercel Analytics for performance
   - Supabase Dashboard for database
   - Sentry for error tracking
   - CloudFlare for CDN analytics

### 11. Migration Path from Current System

1. **Week 1-2**: Set up new database schema
2. **Week 3-4**: Build authentication system
3. **Week 5-6**: Create onboarding flow with preview
4. **Week 7-8**: Implement multi-tenant routing
5. **Week 9-10**: Build new CMS interface
6. **Week 11-12**: Testing and optimization
7. **Launch**: Migrate existing blog as first tenant
8. **Post-Launch**: Open registration to new users

### 12. Cost Estimation

#### Infrastructure (Monthly)
- Vercel Pro: $20
- Supabase Pro: $25
- Domain: $1
- CloudFlare: Free tier
- **Total**: ~$46/month initially

#### Scaling Costs
- Per 1000 active blogs: ~$100-200/month
- Storage: $0.021/GB on Supabase
- Bandwidth: Included in Vercel up to 1TB

### 13. Environment Variables

```env
# Authentication
NEXTAUTH_URL=https://yourdomain.com
NEXTAUTH_SECRET=your-secret-key-here

# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
SUPABASE_SERVICE_KEY=your-service-key

# Email (for verification)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASS=your-app-password

# Platform Config
PLATFORM_DOMAIN=yourdomain.com
DEFAULT_THEME=minimal
MAX_FREE_POSTS=50
MAX_FREE_STORAGE_MB=100
```

### 14. Next Steps

1. **Immediate Actions**
   - Create new Supabase project for multi-tenant
   - Set up development environment
   - Start with authentication implementation

2. **Development Priority**
   - Authentication system
   - Live preview onboarding
   - Basic multi-tenant routing
   - Simple CMS interface
   - Deploy MVP

3. **Testing Strategy**
   - Unit tests for auth logic
   - Integration tests for tenant isolation
   - E2E tests for signup flow
   - Load testing for multiple tenants

This plan provides a complete roadmap for transforming your single-tenant blog into a scalable multi-tenant platform where users can create their own blogs with a delightful onboarding experience.