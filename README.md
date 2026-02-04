## Bem vindo ao Melhor Site de São Paulo! 👋
export interface SMService {
  service: number;
  name: string;
  type: string;
  rate: string;
  min: string;
  max: string;
  dripfeed?: boolean;
  refill?: boolean;
  cancel?: boolean;
  category: string;
}

export interface SMOrder {
  id: string;
  service: number;
  serviceName?: string;
  link: string;
  quantity: number;
  status: 'Pending' | 'In progress' | 'Completed' | 'Partial' | 'Canceled' | 'Processing';
  startCount?: number;
  remains?: number;
  createdAt: Date;
  charge?: string;
}

export interface SMBalance {
  balance: string;
  currency: string;
}

export interface APIConfig {
  apiKey: string;
  apiUrl: string;
}

const DEFAULT_API_URL = 'https://smmapro.com/api/v2';

// Store API config in localStorage
export const getApiConfig = (): APIConfig | null => {
  const config = localStorage.getItem('smm_api_config');
  return config ? JSON.parse(config) : null;
};

export const setApiConfig = (config: APIConfig): void => {
  localStorage.setItem('smm_api_config', JSON.stringify(config));
};

export const clearApiConfig = (): void => {
  localStorage.removeItem('smm_api_config');
};

// API Client
class SMMApiClient {
  private getConfig(): APIConfig {
    const config = getApiConfig();
    if (!config) {
      throw new Error('API não configurada. Por favor, configure sua API Key.');
    }
    return config;
  }

  private async request(action: string, params: Record<string, string | number> = {}): Promise<any> {
    const config = this.getConfig();
    
    const formData = new URLSearchParams();
    formData.append('key', config.apiKey);
    formData.append('action', action);
    
    Object.entries(params).forEach(([key, value]) => {
      formData.append(key, String(value));
    });

    const response = await fetch(config.apiUrl, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded',
      },
      body: formData.toString(),
    });

    const data = await response.json();
    
    if (data.error) {
      throw new Error(data.error);
    }
    
    return data;
  }

  async getServices(): Promise<SMService[]> {
    const services = await this.request('services');
    return Array.isArray(services) ? services : [];
  }

  async getBalance(): Promise<SMBalance> {
    const data = await this.request('balance');
    return {
      balance: data.balance || '0',
      currency: data.currency || 'USD',
    };
  }

  async addOrder(params: {
    service: number;
    link: string;
    quantity: number;
    runs?: number;
    interval?: number;
  }): Promise<{ order: number }> {
    return this.request('add', params as any);
  }

  async getOrderStatus(orderId: number): Promise<any> {
    return this.request('status', { order: orderId });
  }

  async getMultiOrderStatus(orderIds: number[]): Promise<any> {
    return this.request('status', { orders: orderIds.join(',') });
  }

  async cancelOrders(orderIds: number[]): Promise<any> {
    return this.request('cancel', { orders: orderIds.join(',') });
  }

  async refillOrder(orderId: number): Promise<any> {
    return this.request('refill', { order: orderId });
  }
}

export const smmApi = new SMMApiClient();

// Mock data for demo purposes
export const mockServices: SMService[] = [
  { service: 1, name: '👍 Instagram Followers [Real] [Max: 100K]', type: 'Default', rate: '0.50', min: '100', max: '100000', category: 'Instagram Followers', refill: true },
  { service: 2, name: '❤️ Instagram Likes [Premium] [Max: 50K]', type: 'Default', rate: '0.30', min: '50', max: '50000', category: 'Instagram Likes', refill: false },
  // ... mais serviços mockados
];

export const mockOrders: SMOrder[] = [
  { id: '12345', service: 1, serviceName: 'Instagram Followers [Real]', link: 'https://instagram.com/user1', quantity: 1000, status: 'Completed', startCount: 500, remains: 0, createdAt: new Date(Date.now() - 86400000), charge: '5.00' },
  // ... mais pedidos mockados
];
import { useState, useEffect } from 'react';
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { smmApi, getApiConfig, setApiConfig, clearApiConfig, APIConfig, mockServices, mockOrders, SMService, SMOrder } from '@/lib/api';

export const useApiConfig = () => {
  const [config, setConfig] = useState<APIConfig | null>(null);
  const [isConfigured, setIsConfigured] = useState(false);

  useEffect(() => {
    const savedConfig = getApiConfig();
    setConfig(savedConfig);
    setIsConfigured(!!savedConfig);
  }, []);

  const saveConfig = (newConfig: APIConfig) => {
    setApiConfig(newConfig);
    setConfig(newConfig);
    setIsConfigured(true);
  };

  const clearConfig = () => {
    clearApiConfig();
    setConfig(null);
    setIsConfigured(false);
  };

  return { config, isConfigured, saveConfig, clearConfig };
};

export const useServices = () => {
  const { isConfigured } = useApiConfig();

  return useQuery({
    queryKey: ['services'],
    queryFn: async () => {
      if (!isConfigured) return mockServices;
      try {
        return await smmApi.getServices();
      } catch {
        return mockServices;
      }
    },
    staleTime: 5 * 60 * 1000,
  });
};

export const useBalance = () => {
  const { isConfigured } = useApiConfig();

  return useQuery({
    queryKey: ['balance'],
    queryFn: async () => {
      if (!isConfigured) return { balance: '125.50', currency: 'USD' };
      try {
        return await smmApi.getBalance();
      } catch {
        return { balance: '0.00', currency: 'USD' };
      }
    },
    refetchInterval: 30000,
  });
};

export const useOrders = () => {
  const [orders, setOrders] = useState<SMOrder[]>(mockOrders);

  const addOrder = (order: Omit<SMOrder, 'id' | 'createdAt'>) => {
    const newOrder: SMOrder = {
      ...order,
      id: String(Date.now()),
      createdAt: new Date(),
    };
    setOrders(prev => [newOrder, ...prev]);
    return newOrder;
  };

  return { orders, addOrder };
};

export const useAddOrder = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (params: { service: number; link: string; quantity: number }) => {
      return smmApi.addOrder(params);
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['balance'] });
    },
  });
};
import { ReactNode, useState } from 'react';
import { motion, AnimatePresence } from 'framer-motion';
import { Link, useLocation } from 'react-router-dom';
import { LayoutDashboard, ShoppingCart, Package, Settings, Menu, X, Zap, ChevronRight } from 'lucide-react';
import { cn } from '@/lib/utils';
import { Button } from '@/components/ui/button';

interface LayoutProps {
  children: ReactNode;
}

const navItems = [
  { icon: LayoutDashboard, label: 'Dashboard', path: '/' },
  { icon: Package, label: 'Serviços', path: '/services' },
  { icon: ShoppingCart, label: 'Novo Pedido', path: '/order' },
  { icon: Settings, label: 'Configurações', path: '/settings' },
];

export const Layout = ({ children }: LayoutProps) => {
  const location = useLocation();
  const [isSidebarOpen, setIsSidebarOpen] = useState(false);

  return (
    <div className="min-h-screen bg-background">
      {/* Mobile Header */}
      <header className="lg:hidden fixed top-0 left-0 right-0 z-50 glass-card border-b border-border/50 px-4 py-3">
        <div className="flex items-center justify-between">
          <Link to="/" className="flex items-center gap-2">
            <div className="w-8 h-8 rounded-lg bg-gradient-to-br from-primary to-accent flex items-center justify-center">
              <Zap className="w-5 h-5 text-primary-foreground" />
            </div>
            <span className="font-display font-bold text-lg gradient-text">SMMPro</span>
          </Link>
          <Button variant="ghost" size="icon" onClick={() => setIsSidebarOpen(!isSidebarOpen)}>
            {isSidebarOpen ? <X className="w-5 h-5" /> : <Menu className="w-5 h-5" />}
          </Button>
        </div>
      </header>

      {/* Sidebar */}
      <aside className={cn(
        'fixed top-0 left-0 h-full w-64 glass-card border-r border-border/50 z-50 transition-transform duration-300',
        'lg:translate-x-0',
        isSidebarOpen ? 'translate-x-0' : '-translate-x-full'
      )}>
        {/* Logo, Navigation, Help Card */}
      </aside>

      {/* Main Content */}
      <main className="lg:ml-64 pt-16 lg:pt-0 min-h-screen">
        <div className="p-4 lg:p-8">
          <motion.div key={location.pathname} initial={{ opacity: 0, y: 20 }} animate={{ opacity: 1, y: 0 }}>
            {children}
          </motion.div>
        </div>
      </main>
    </div>
  );
};
import { motion } from 'framer-motion';
import { LucideIcon } from 'lucide-react';
import { cn } from '@/lib/utils';

interface StatCardProps {
  title: string;
  value: string | number;
  subtitle?: string;
  icon: LucideIcon;
  trend?: { value: number; isPositive: boolean };
  variant?: 'default' | 'primary' | 'success' | 'warning';
  delay?: number;
}

export const StatCard = ({ title, value, subtitle, icon: Icon, trend, variant = 'default', delay = 0 }: StatCardProps) => {
  return (
    <motion.div initial={{ opacity: 0, y: 20 }} animate={{ opacity: 1, y: 0 }} transition={{ duration: 0.4, delay }} className={cn('stat-card bg-gradient-to-br', variantStyles[variant])}>
      <div className="flex items-start justify-between">
        <div>
          <p className="text-sm text-muted-foreground font-medium">{title}</p>
          <h3 className="text-3xl font-display font-bold mt-2 text-foreground">{value}</h3>
          {subtitle && <p className="text-xs text-muted-foreground mt-1">{subtitle}</p>}
          {trend && (
            <div className="flex items-center gap-1 mt-2">
              <span className={cn('text-xs font-medium', trend.isPositive ? 'text-success' : 'text-destructive')}>
                {trend.isPositive ? '+' : ''}{trend.value}%
              </span>
            </div>
          )}
        </div>
        <div className={cn('p-3 rounded-xl', iconVariantStyles[variant])}>
          <Icon className="w-6 h-6" />
        </div>
      </div>
    </motion.div>
  );
};
import { Toaster } from "@/components/ui/toaster";
import { Toaster as Sonner } from "@/components/ui/sonner";
import { TooltipProvider } from "@/components/ui/tooltip";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { BrowserRouter, Routes, Route } from "react-router-dom";
import Index from "./pages/Index";
import Services from "./pages/Services";
import Order from "./pages/Order";
import Settings from "./pages/Settings";
import NotFound from "./pages/NotFound";

const queryClient = new QueryClient();

const App = () => (
  <QueryClientProvider client={queryClient}>
    <TooltipProvider>
      <Toaster />
      <Sonner position="top-right" theme="dark" />
      <BrowserRouter>
        <Routes>
          <Route path="/" element={<Index />} />
          <Route path="/services" element={<Services />} />
          <Route path="/order" element={<Order />} />
          <Route path="/settings" element={<Settings />} />
          <Route path="*" element={<NotFound />} />
        </Routes>
      </BrowserRouter>
    </TooltipProvider>
  </QueryClientProvider>
);

export default App;

<!--
**AGENCIAPREMIUMH/agenciapremiumh** is a ✨ _special_ ✨ repository because its `README.md` (this file) appears on your GitHub profile.

Here are some ideas to get you started:

- 🔭 I’m currently working on ...
- 🌱 I’m currently learning ...
- 👯 I’m looking to collaborate on ...
- 🤔 I’m looking for help with ...
- 💬 Ask me about ...
- 📫 How to reach me: ...
- 😄 Pronouns: ...
- ⚡ Fun fact: ...
-->
