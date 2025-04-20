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

```tsx
'use client';

import { pluginRegistry, HOOKS } from '@/lib/plugins/registry';
import { useState, useEffect } from 'react';

export default function Panel({ selectedElement, onChange }: {
  selectedElement: any;
  onChange: (newProps: any) => void;
}) {
  const [controls, setControls] = useState<React.ReactNode[]>([]);

  useEffect(() => {
    if (!selectedElement) return;

    const blockType = pluginRegistry.getBlock(selectedElement.type);
    const newControls: React.ReactNode[] = [];

    // Add built-in controls
    if (blockType?.panelControls) {
      const PanelControls = blockType.panelControls;
      newControls.push(
        <PanelControls 
          key="built-in" 
          {...selectedElement.props} 
          onChange={onChange}
        />
      );
    }

    // Add plugin controls
    const pluginControls = pluginRegistry.runHook(
      HOOKS.ELEMENT_CONTROLS, 
      selectedElement.type
    );
    
    pluginControls.forEach((Control, index) => {
      if (typeof Control === 'function') {
        newControls.push(
          <Control 
            key={`plugin-${index}`}
            {...selectedElement.props}
            onChange={onChange}
          />
        );
      }
    });

    setControls(newControls);
  }, [selectedElement, onChange]);

  if (!selectedElement) {
    return (
      <div className="panel-empty">
        <p>Select an element to edit its properties</p>
      </div>
    );
  }

  const blockType = pluginRegistry.getBlock(selectedElement.type);

  return (
    <div className="builder-panel">
      <h3>{blockType?.name || 'Element'} Settings</h3>
      <div className="panel-controls">
        {controls}
      </div>
    </div>
  );
}
```

Element Component (components/builder/Element.tsx)

```tsx
'use client';

import { useSortable } from '@dnd-kit/sortable';
import { CSS } from '@dnd-kit/utilities';

export default function Element({ 
  id, 
  children, 
  onSelect 
}: {
  id: string;
  children: React.ReactNode;
  onSelect: () => void;
}) {
  const {
    attributes,
    listeners,
    setNodeRef,
    transform,
    transition,
    isDragging,
  } = useSortable({ id });

  const style = {
    transform: CSS.Transform.toString(transform),
    transition,
    opacity: isDragging ? 0.5 : 1,
    cursor: 'move',
    position: 'relative',
  };

  return (
    <div
      ref={setNodeRef}
      style={style}
      {...attributes}
      {...listeners}
      onClick={(e) => {
        e.stopPropagation();
        onSelect();
      }}
      className="builder-element"
    >
      {children}
      <div className="element-toolbar">
        <button>Delete</button>
        <button>Duplicate</button>
      </div>
    </div>
  );
}
```
 
  
## 6. Example Plugins

  Basic Blocks Plugin (plugins/basic-blocks.ts)

```tsx
import React from 'react';
import { FaHeading, FaParagraph, FaImage, FaSquare } from 'react-icons/fa';
import { HOOKS } from '@/lib/plugins/hooks';
import { pluginRegistry } from '@/lib/plugins/registry';

// Heading Component
const HeadingBlock = ({ text, level = 2, onChange }: any) => {
  const Tag = `h${Math.min(Math.max(level, 1), 6)}` as keyof JSX.IntrinsicElements;
  
  return (
    <Tag
      contentEditable
      suppressContentEditableWarning
      style={{ fontSize: `${2.5 - (level * 0.2)}rem` }}
      onBlur={(e) => onChange({ text: e.target.innerText })}
    >
      {text}
    </Tag>
  );
};

const HeadingControls = ({ level, onChange }: any) => (
  <div>
    <label>Heading Level</label>
    <select 
      value={level} 
      onChange={(e) => onChange({ level: parseInt(e.target.value) })}
    >
      {[1, 2, 3, 4, 5, 6].map(lvl => (
        <option key={lvl} value={lvl}>H{lvl}</option>
      ))}
    </select>
  </div>
);

// Paragraph Component
const ParagraphBlock = ({ text, onChange }: any) => (
  <p
    contentEditable
    suppressContentEditableWarning
    onBlur={(e) => onChange({ text: e.target.innerText })}
  >
    {text}
  </p>
);

// Image Component
const ImageBlock = ({ src, alt, onChange }: any) => (
  <div>
    {src ? (
      <img src={src} alt={alt} style={{ maxWidth: '100%' }} />
    ) : (
      <div className="image-placeholder">Select an image</div>
    )}
    <button onClick={() => {/* Open media library */}}>
      Select Image
    </button>
  </div>
);

const ImageControls = ({ alt, onChange }: any) => (
  <div>
    <label>Alt Text</label>
    <input
      type="text"
      value={alt}
      onChange={(e) => onChange({ alt: e.target.value })}
    />
  </div>
);

// Section Component
const SectionBlock = ({ children, background, padding }: any) => (
  <div style={{ background, padding }}>
    {children}
  </div>
);

const SectionControls = ({ background, padding, onChange }: any) => (
  <div>
    <label>Background</label>
    <input
      type="color"
      value={background}
      onChange={(e) => onChange({ background: e.target.value })}
    />
    
    <label>Padding</label>
    <input
      type="text"
      value={padding}
      onChange={(e) => onChange({ padding: e.target.value })}
    />
  </div>
);

// Register the plugin
export default {
  name: 'basic-blocks',
  version: '1.0.0',
  blocks: [
    {
      id: 'heading',
      name: 'Heading',
      icon: <FaHeading />,
      component: HeadingBlock,
      panelControls: HeadingControls,
      defaultProps: {
        text: 'New Heading',
        level: 2
      }
    },
    {
      id: 'paragraph',
      name: 'Paragraph',
      icon: <FaParagraph />,
      component: ParagraphBlock,
      defaultProps: {
        text: 'Lorem ipsum dolor sit amet, consectetur adipiscing elit.'
      }
    },
    {
      id: 'image',
      name: 'Image',
      icon: <FaImage />,
      component: ImageBlock,
      panelControls: ImageControls,
      defaultProps: {
        src: '',
        alt: ''
      }
    },
    {
      id: 'section',
      name: 'Section',
      icon: <FaSquare />,
      component: SectionBlock,
      panelControls: SectionControls,
      defaultProps: {
        background: '#ffffff',
        padding: '20px',
        children: []
      }
    }
  ],
  hooks: {
    [HOOKS.BEFORE_SAVE]: [(content: any) => {
      // Sanitize content before saving
      return {
        ...content,
        elements: content.elements.map((el: any) => ({
          ...el,
          props: sanitizeProps(el.props)
        }))
      };
    }]
  }
};

function sanitizeProps(props: any) {
  // Implement proper sanitization based on your needs
  return props;
}
```

Advanced Blocks Plugin (plugins/advanced-blocks.ts)

```tsx
import React from 'react';
import { FaButton, FaColumns, FaVideo } from 'react-icons/fa';
import { HOOKS } from '@/lib/plugins/hooks';
import { pluginRegistry } from '@/lib/plugins/registry';

// Button Component
const ButtonBlock = ({ text, url, style, onChange }: any) => (
  <a 
    href={url} 
    style={{
      display: 'inline-block',
      padding: '10px 20px',
      background: style?.background || '#0070f3',
      color: style?.color || 'white',
      borderRadius: style?.radius || '4px',
      textDecoration: 'none'
    }}
  >
    {text}
  </a>
);

const ButtonControls = ({ text, url, style, onChange }: any) => (
  <div>
    <label>Button Text</label>
    <input
      type="text"
      value={text}
      onChange={(e) => onChange({ text: e.target.value })}
    />
    
    <label>Button URL</label>
    <input
      type="text"
      value={url}
      onChange={(e) => onChange({ url: e.target.value })}
    />
    
    <label>Background Color</label>
    <input
      type="color"
      value={style?.background || '#0070f3'}
      onChange={(e) => onChange({ 
        style: { ...style, background: e.target.value } 
      })}
    />
  </div>
);

// Columns Component
const ColumnsBlock = ({ columns = 2, children, onChange }: any) => {
  const columnStyle = {
    flex: `1 0 ${100 / columns}%`,
    padding: '10px'
  };
  
  return (
    <div style={{ display: 'flex' }}>
      {Array.from({ length: columns }).map((_, i) => (
        <div key={i} style={columnStyle}>
          {children?.[i] || <div>Column {i + 1}</div>}
        </div>
      ))}
    </div>
  );
};

const ColumnsControls = ({ columns, onChange }: any) => (
  <div>
    <label>Number of Columns</label>
    <select
      value={columns}
      onChange={(e) => onChange({ columns: parseInt(e.target.value) })}
    >
      {[1, 2, 3, 4].map(num => (
        <option key={num} value={num}>{num}</option>
      ))}
    </select>
  </div>
);

// Register the plugin
export default {
  name: 'advanced-blocks',
  version: '1.0.0',
  blocks: [
    {
      id: 'button',
      name: 'Button',
      icon: <FaButton />,
      component: ButtonBlock,
      panelControls: ButtonControls,
      defaultProps: {
        text: 'Click Me',
        url: '#',
        style: {
          background: '#0070f3',
          color: 'white',
          radius: '4px'
        }
      }
    },
    {
      id: 'columns',
      name: 'Columns',
      icon: <FaColumns />,
      component: ColumnsBlock,
      panelControls: ColumnsControls,
      defaultProps: {
        columns: 2,
        children: []
      }
    },
    {
      id: 'video',
      name: 'Video',
      icon: <FaVideo />,
      component: VideoBlock,
      panelControls: VideoControls,
      defaultProps: {
        url: '',
        autoplay: false,
        controls: true
      }
    }
  ],
  hooks: {
    [HOOKS.TOOLBAR_EXTEND]: [() => {
      return (
        <div className="toolbar-section">
          <h4>Advanced</h4>
        </div>
      );
    }]
  }
};

// Video Component (implementation omitted for brevity)
function VideoBlock(props: any) {
  return <div>Video Block</div>;
}

function VideoControls(props: any) {
  return <div>Video Controls</div>;
}
```




## 7. Page Builder Page (app/(builder)/page/[id]/page.tsx)

```tsx
import { prisma } from '@/lib/prisma';
import { loadPlugins } from '@/lib/plugins/loader';
import Canvas from '@/components/builder/Canvas';
import Toolbar from '@/components/builder/Toolbar';
import Panel from '@/components/builder/Panel';
import { useState } from 'react';

export default async function PageBuilder({ params }: { params: { id: string } }) {
  // Load plugins (this would ideally happen at app startup)
  await loadPlugins();
  
  const pageData = await prisma.page.findUnique({
    where: { id: parseInt(params.id) },
    include: {
      revisions: {
        orderBy: { createdAt: 'desc' },
        take: 1
      }
    }
  });
  
  if (!pageData) return <div>Page not found</div>;
  
  const initialContent = pageData.revisions[0]?.content || { elements: [] };

  return (
    <BuilderApp initialContent={initialContent} pageId={params.id} />
  );
}

function BuilderApp({ initialContent, pageId }: { 
  initialContent: any; 
  pageId: string;
}) {
  const [content, setContent] = useState(initialContent);
  const [selectedElement, setSelectedElement] = useState<any>(null);

  const handleUpdate = async (newContent: any) => {
    setContent(newContent);
    
    // Auto-save implementation
    try {
      const response = await fetch(`/api/builder/pages?id=${pageId}`, {
        method: 'PUT',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          id: pageId,
          title: content.title || 'Untitled',
          content: newContent
        })
      });
      
      if (!response.ok) {
        console.error('Failed to save page');
      }
    } catch (error) {
      console.error('Error saving page:', error);
    }
  };

  return (
    <div className="builder-container">
      <div className="builder-header">
        <h1>Page Builder</h1>
        <button>Save</button>
        <button>Preview</button>
      </div>
      
      <div className="builder-layout">
        <Toolbar />
        
        <div className="builder-main">
          <Canvas 
            initialData={content} 
            onUpdate={handleUpdate}
          />
        </div>
        
        <Panel 
          selectedElement={selectedElement}
          onChange={(newProps) => {
            if (selectedElement) {
              handleElementChange(selectedElement.id, newProps);
            }
          }}
        />
      </div>
    </div>
  );
}
```



## 8. CSS Styles
```css
/* styles/builder.css */
.builder-container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  overflow: hidden;
}

.builder-header {
  padding: 1rem;
  background: #1a1a1a;
  color: white;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.builder-layout {
  display: flex;
  flex: 1;
  overflow: hidden;
}

.builder-toolbar {
  width: 250px;
  background: #f5f5f5;
  padding: 1rem;
  overflow-y: auto;
}

.builder-main {
  flex: 1;
  padding: 1rem;
  overflow: auto;
  background: #f9f9f9;
}

.builder-panel {
  width: 300px;
  background: white;
  padding: 1rem;
  border-left: 1px solid #ddd;
  overflow-y: auto;
}

.elements-list {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 0.5rem;
}

.element-item {
  padding: 0.5rem;
  background: white;
  border: 1px solid #ddd;
  border-radius: 4px;
  cursor: grab;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.element-item:hover {
  background: #f0f0f0;
}

.element-icon {
  font-size: 1.2rem;
}

.builder-canvas {
  min-height: 100%;
  padding: 2rem;
  outline: none;
}

.builder-element {
  margin-bottom: 1rem;
  border: 1px dashed transparent;
  padding: 0.5rem;
}

.builder-element:hover {
  border-color: #0070f3;
}

.element-toolbar {
  position: absolute;
  top: -30px;
  right: 0;
  background: #0070f3;
  padding: 0.25rem;
  border-radius: 4px 4px 0 0;
  display: none;
}

.builder-element:hover .element-toolbar {
  display: flex;
  gap: 0.25rem;
}

.panel-controls {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.panel-controls label {
  display: block;
  margin-bottom: 0.25rem;
  font-weight: 500;
}

.panel-controls input,
.panel-controls select {
  width: 100%;
  padding: 0.5rem;
  border: 1px solid #ddd;
  border-radius: 4px;
}
```



## 9. Environment Variables (.env)

```env
DATABASE_URL="postgresql://user:password@localhost:5432/pagebuilder?schema=public"
NEXT_PUBLIC_SUPABASE_URL=your_supabase_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_key
```

### Implementation Notes

1. Dependencies: Make sure to install all required packages:
```bash
npm install @dnd-kit/core @dnd-kit/sortable @dnd-kit/utables @prisma/client react-icons
```
2. Database Setup: Run Prisma migrations after setting up your schema:
```bash
   npx prisma migrate dev --name init
```
3. Authentication: You'll want to add authentication (NextAuth.js is a good choice) to protect your builder routes.
4. Production Considerations:
  - Implement proper error boundaries 
  - Add loading states 
  - Consider adding undo/redo functionality 
  - Implement proper sanitization for HTML content

5. Extending:
  - Add more block types via plugins 
  - Implement template saving/loading 
  - Add responsive design controls 
  - Implement global style settings


