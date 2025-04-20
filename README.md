#  Elementor-like Page Builder with Next.js, Prisma, and Plugin System
I am writing to build Elementor-like Page Builder with Next.js, Prisma, and Plugin System

## 1. Project Setup
  File Stracture

```bash
├── app/
│   ├── (builder)/
│   │   ├── page/
│   │   │   └── [id]/
│   │   │       └── page.tsx
│   ├── api/
│   │   ├── builder/
│   │   │   ├── pages/
│   │   │   │   └── route.ts
│   │   │   └── plugins/
│   │   │       └── route.ts
│   ├── components/
│   │   └── builder/
│   │       ├── Canvas.tsx
│   │       ├── Toolbar.tsx
│   │       ├── Panel.tsx
│   │       └── Element.tsx
├── lib/
│   ├── plugins/
│   │   ├── registry.ts
│   │   ├── hooks.ts
│   │   └── loader.ts
│   └── prisma.ts
├── plugins/
│   ├── basic-blocks.ts
│   └── advanced-blocks.ts
├── prisma/
│   └── schema.prisma
└── .env
```




## 2. Core Files

  Prisma Setup (lib/prisma.ts)

```ts
import { PrismaClient } from '@prisma/client';

declare global {
  var prisma: PrismaClient | undefined;
}

const prisma = globalThis.prisma || new PrismaClient();

if (process.env.NODE_ENV !== 'production') globalThis.prisma = prisma;

export default prisma;
```

Plugin Registry (lib/plugins/registry.ts)

```ts
import React from 'react';

type BlockType = {
  id: string;
  name: string;
  icon: React.ReactNode;
  component: React.ComponentType<any>;
  panelControls?: React.ComponentType<{ 
    [key: string]: any;
    onChange: (props: any) => void 
  }>;
  defaultProps?: Record<string, any>;
};

type Plugin = {
  name: string;
  version?: string;
  blocks?: BlockType[];
  hooks?: Record<string, Function[]>;
};

class PluginRegistry {
  private plugins: Plugin[] = [];
  private blocks: Record<string, BlockType> = {};
  private hooks: Record<string, Function[]> = {};

  register(plugin: Plugin) {
    this.plugins.push(plugin);
    
    // Register blocks
    plugin.blocks?.forEach(block => {
      this.blocks[block.id] = {
        ...block,
        defaultProps: block.defaultProps || {}
      };
    });
    
    // Register hooks
    if (plugin.hooks) {
      for (const [hookName, callbacks] of Object.entries(plugin.hooks)) {
        this.hooks[hookName] = [...(this.hooks[hookName] || []), ...callbacks];
      }
    }
  }

  getBlock(id: string): BlockType | undefined {
    return this.blocks[id];
  }

  getBlocks(): BlockType[] {
    return Object.values(this.blocks);
  }

  runHook<T = any>(hookName: string, ...args: any[]): T[] {
    const callbacks = this.hooks[hookName] || [];
    return callbacks.map(callback => callback(...args));
  }
}

export const pluginRegistry = new PluginRegistry();
```


Hooks Definition (lib/plugins/hooks.ts)
```ts
export const HOOKS = {
  BEFORE_SAVE: 'before_save',
  AFTER_SAVE: 'after_save',
  BEFORE_RENDER: 'before_render',
  TOOLBAR_EXTEND: 'toolbar_extend',
  PANEL_EXTEND: 'panel_extend',
  ELEMENT_CONTROLS: 'element_controls',
  ELEMENT_RENDER: 'element_render',
} as const;
```


Plugin Loader (lib/plugins/loader.ts)

```ts
import { pluginRegistry } from './registry';

export async function loadPlugins() {
  // Load built-in plugins
  const builtInPlugins = [
    await import('@/plugins/basic-blocks'),
    await import('@/plugins/advanced-blocks'),
  ];

  builtInPlugins.forEach(plugin => {
    if (plugin.default) {
      pluginRegistry.register(plugin.default);
    }
  });

  try {
    // Load dynamic plugins from API
    const response = await fetch('/api/plugins');
    if (response.ok) {
      const plugins = await response.json();
      
      for (const plugin of plugins) {
        if (plugin.isActive) {
          try {
            const module = await import(`/plugins/${plugin.name}`);
            pluginRegistry.register(module.default);
          } catch (error) {
            console.error(`Failed to load plugin ${plugin.name}:`, error);
          }
        }
      }
    }
  } catch (error) {
    console.error('Failed to fetch plugins:', error);
  }
}
```
 

## 3. Prisma Schema (prisma/schema.prisma)
```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id            Int       @id @default(autoincrement())
  email         String    @unique
  passwordHash  String
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
  pages         Page[]
  elements      Element[]
  media         Media[]
  plugins       Plugin[]
}

model Page {
  id            Int       @id @default(autoincrement())
  title         String
  slug          String    @unique
  status        String    @default("draft")
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
  user          User      @relation(fields: [userId], references: [id])
  userId        Int
  revisions     PageRevision[]
}

model PageRevision {
  id            Int       @id @default(autoincrement())
  content       Json
  createdAt     DateTime  @default(now())
  page          Page      @relation(fields: [pageId], references: [id])
  pageId        Int
}

model Element {
  id            Int       @id @default(autoincrement())
  name          String
  type          String
  content       Json
  createdAt     DateTime  @default(now())
  user          User      @relation(fields: [userId], references: [id])
  userId        Int
}

model Media {
  id            Int       @id @default(autoincrement())
  url           String
  fileName      String
  fileType      String
  fileSize      Int
  createdAt     DateTime  @default(now())
  user          User      @relation(fields: [userId], references: [id])
  userId        Int
}

model Plugin {
  id          Int      @id @default(autoincrement())
  name        String   @unique
  version     String
  isActive    Boolean  @default(false)
  settings    Json?
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  user        User?    @relation(fields: [userId], references: [id])
  userId      Int?
}
```
 

## 4. API Routes
  Pages API (app/api/builder/pages/route.ts)

```ts
import { NextResponse } from 'next/server';
import prisma from '@/lib/prisma';
import { pluginRegistry, HOOKS } from '@/lib/plugins/registry';

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const pageId = searchParams.get('id');
  
  try {
    if (pageId) {
      const page = await prisma.page.findUnique({
        where: { id: parseInt(pageId) },
        include: {
          revisions: {
            orderBy: { createdAt: 'desc' },
            take: 1
          }
        }
      });
      
      if (!page) {
        return NextResponse.json({ error: 'Page not found' }, { status: 404 });
      }
      
      // Run before_render hooks
      const content = page.revisions[0]?.content || { elements: [] };
      pluginRegistry.runHook(HOOKS.BEFORE_RENDER, content);
      
      return NextResponse.json({
        ...page,
        content
      });
    } else {
      const pages = await prisma.page.findMany({
        select: {
          id: true,
          title: true,
          slug: true,
          status: true,
          updatedAt: true
        }
      });
      
      return NextResponse.json(pages);
    }
  } catch (error) {
    return NextResponse.json(
      { error: 'Database error', details: error instanceof Error ? error.message : String(error) },
      { status: 500 }
    );
  }
}

export async function POST(request: Request) {
  try {
    const { title, userId, content } = await request.json();
    
    // Run before_save hooks
    const processedContent = pluginRegistry.runHook(HOOKS.BEFORE_SAVE, content)
      .reduce((acc, result) => ({ ...acc, ...result }), content);
    
    const slug = generateSlug(title);
    
    const page = await prisma.page.create({
      data: {
        title,
        slug,
        user: { connect: { id: userId } },
        revisions: {
          create: { content: processedContent }
        }
      },
      include: {
        revisions: {
          orderBy: { createdAt: 'desc' },
          take: 1
        }
      }
    });
    
    // Run after_save hooks
    pluginRegistry.runHook(HOOKS.AFTER_SAVE, page);
    
    return NextResponse.json({
      ...page,
      content: page.revisions[0]?.content
    });
  } catch (error) {
    return NextResponse.json(
      { error: 'Failed to create page', details: error instanceof Error ? error.message : String(error) },
      { status: 500 }
    );
  }
}

export async function PUT(request: Request) {
  try {
    const { id, title, content } = await request.json();
    
    // Run before_save hooks
    const processedContent = pluginRegistry.runHook(HOOKS.BEFORE_SAVE, content)
      .reduce((acc, result) => ({ ...acc, ...result }), content);
    
    const [updatedPage] = await prisma.$transaction([
      prisma.page.update({
        where: { id },
        data: { title, updatedAt: new Date() }
      }),
      prisma.pageRevision.create({
        data: {
          content: processedContent,
          page: { connect: { id } }
        }
      })
    ]);
    
    // Run after_save hooks
    pluginRegistry.runHook(HOOKS.AFTER_SAVE, updatedPage);
    
    return NextResponse.json({ success: true, page: updatedPage });
  } catch (error) {
    return NextResponse.json(
      { error: 'Failed to update page', details: error instanceof Error ? error.message : String(error) },
      { status: 500 }
    );
  }
}

function generateSlug(title: string) {
  return title
    .toLowerCase()
    .replace(/[^\w\s-]/g, '')
    .replace(/[\s_-]+/g, '-')
    .replace(/^-+|-+$/g, '');
}
```

Plugins API (app/api/plugins/route.ts)

```ts
import { NextResponse } from 'next/server';
import prisma from '@/lib/prisma';

export async function GET() {
  try {
    const plugins = await prisma.plugin.findMany({
      where: { isActive: true },
      select: {
        id: true,
        name: true,
        version: true,
        isActive: true,
        settings: true
      }
    });
    
    return NextResponse.json(plugins);
  } catch (error) {
    return NextResponse.json(
      { error: 'Failed to fetch plugins', details: error instanceof Error ? error.message : String(error) },
      { status: 500 }
    );
  }
}

export async function POST(request: Request) {
  try {
    const { name, version, settings } = await request.json();
    
    const plugin = await prisma.plugin.upsert({
      where: { name },
      update: { 
        version,
        settings,
        isActive: true,
        updatedAt: new Date()
      },
      create: {
        name,
        version,
        settings,
        isActive: true
      }
    });
    
    return NextResponse.json(plugin);
  } catch (error) {
    return NextResponse.json(
      { error: 'Failed to activate plugin', details: error instanceof Error ? error.message : String(error) },
      { status: 500 }
    );
  }
}
```
 

## 5. Builder Components
  Canvas Component (components/builder/Canvas.tsx)

```ts
'use client';

import { useState, useCallback } from 'react';
import { DndContext, DragOverlay, closestCenter } from '@dnd-kit/core';
import { arrayMove, SortableContext } from '@dnd-kit/sortable';
import { pluginRegistry, HOOKS } from '@/lib/plugins/registry';
import Element from './Element';

export default function Canvas({ initialData, onUpdate }: {
  initialData: any;
  onUpdate: (data: any) => void;
}) {
  const [elements, setElements] = useState(initialData.elements || []);
  const [activeElement, setActiveElement] = useState<any>(null);

  const handleDragStart = useCallback((event: any) => {
    if (event.active.data.current?.type === 'element') {
      setActiveElement({
        id: `el-${Date.now()}`,
        type: event.active.data.current.elementType,
        props: event.active.data.current.defaultProps || {}
      });
    } else {
      setActiveElement(event.active.data.current?.element);
    }
  }, []);

  const handleDragEnd = useCallback((event: any) => {
    const { active, over } = event;
    
    if (active.id !== over?.id && activeElement) {
      setElements((items: any[]) => {
        const newElements = [...items, activeElement];
        onUpdate({ ...initialData, elements: newElements });
        return newElements;
      });
    } else if (active.id !== over.id) {
      setElements((items: any[]) => {
        const oldIndex = items.findIndex(item => item.id === active.id);
        const newIndex = items.findIndex(item => item.id === over.id);
        const newElements = arrayMove(items, oldIndex, newIndex);
        onUpdate({ ...initialData, elements: newElements });
        return newElements;
      });
    }
    
    setActiveElement(null);
  }, [activeElement, initialData, onUpdate]);

  const handleElementChange = useCallback((id: string, newProps: any) => {
    setElements((currentElements: any[]) => {
      const newElements = currentElements.map(el => 
        el.id === id ? { ...el, props: { ...el.props, ...newProps } } : el
      );
      onUpdate({ ...initialData, elements: newElements });
      return newElements;
    });
  }, [initialData, onUpdate]);

  const renderElement = (element: any) => {
    const blockType = pluginRegistry.getBlock(element.type);
    if (!blockType) return <div>Unknown element type: {element.type}</div>;
    
    const BlockComponent = blockType.component;
    
    // Run element_render hooks
    const renderProps = pluginRegistry.runHook(
      HOOKS.ELEMENT_RENDER, 
      element.props, 
      element
    ).reduce((props, hookResult) => ({ ...props, ...hookResult }), element.props);
    
    return (
      <Element 
        id={element.id} 
        onSelect={() => {/* handle selection */}}
      >
        <BlockComponent 
          {...renderProps}
          onChange={(newProps: any) => handleElementChange(element.id, newProps)}
        />
      </Element>
    );
  };

  return (
    <DndContext
      collisionDetection={closestCenter}
      onDragStart={handleDragStart}
      onDragEnd={handleDragEnd}
    >
      <SortableContext items={elements}>
        <div className="builder-canvas">
          {elements.map((element: any) => (
            <div key={element.id}>
              {renderElement(element)}
            </div>
          ))}
        </div>
      </SortableContext>
      
      <DragOverlay>
        {activeElement ? renderElement(activeElement) : null}
      </DragOverlay>
    </DndContext>
  );
}
```

Toolbar Component (components/builder/Toolbar.tsx)

```ts
'use client';

import { useDraggable } from '@dnd-kit/core';
import { pluginRegistry } from '@/lib/plugins/registry';

export default function Toolbar() {
  const blocks = pluginRegistry.getBlocks();

  return (
    <div className="builder-toolbar">
      <h3>Elements</h3>
      <div className="elements-list">
        {blocks.map((block) => (
          <DraggableElement key={block.id} block={block} />
        ))}
      </div>
    </div>
  );
}

function DraggableElement({ block }: { block: any }) {
  const { attributes, listeners, setNodeRef } = useDraggable({
    id: `new-${block.id}-${Date.now()}`,
    data: {
      type: 'element',
      elementType: block.id,
      defaultProps: block.defaultProps || {}
    },
  });

  return (
    <div
      ref={setNodeRef}
      {...attributes}
      {...listeners}
      className="element-item"
    >
      <span className="element-icon">{block.icon}</span>
      {block.name}
    </div>
  );
}
```


Panel Component (components/builder/Panel.tsx)



  
## 6. Example Plugins
## 7. Page Builder Page (app/(builder)/page/[id]/page.tsx)
## 8. CSS Styles
## 9. Environment Variables (.env)
