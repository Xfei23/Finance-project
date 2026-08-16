# 1. Scaffold the Next.js app
npx create-next-app@14 financial-companion --typescript --tailwind --eslint --app --src-dir=false --import-alias="@/*" --use-npm --yes

# 2. Enter the directory and install extra dependencies
cd financial-companion
npm install zustand recharts lucide-react date-fns

# 3. Create required directories
mkdir -p lib components

# 4. Write all application files
echo "Writing files..."

cat << 'EOF' > tailwind.config.ts
import type { Config } from "tailwindcss";

const config: Config = {
  content: [
    "./pages/**/*.{js,ts,jsx,tsx,mdx}",
    "./components/**/*.{js,ts,jsx,tsx,mdx}",
    "./app/**/*.{js,ts,jsx,tsx,mdx}",
  ],
  theme: {
    extend: {
      colors: {
        brand: {
          50: '#eff6ff',
          100: '#dbeafe',
          500: '#3b82f6',
          600: '#2563eb',
          900: '#1e3a8a',
        }
      }
    },
  },
  plugins: [],
};
export default config;
EOF

cat << 'EOF' > app/globals.css
@tailwind base;
@tailwind components;
@tailwind utilities;

body {
  background-color: #f8fafc;
  color: #0f172a;
}
EOF

cat << 'EOF' > lib/types.ts
export type AssetCategory = 
  | 'Stocks' 
  | 'ETFs' 
  | 'Crypto' 
  | 'RealEstate' 
  | 'Bonds' 
  | 'PreciousMetals' 
  | 'IntlEquities';

export interface Holding {
  id: string;
  category: AssetCategory;
  ticker: string;
  name: string;
  quantity: number;
  purchasePrice: number;
}

export interface PriceData {
  ticker: string;
  currentPrice: number;
  change24h: number;
}

export interface NewsArticle {
  title: string;
  description: string;
  url: string;
  publishedAt: string;
  source: { name: string };
  relatedCategories: AssetCategory[];
}
EOF

cat << 'EOF' > lib/education.ts
import { AssetCategory } from './types';

export const CATEGORY_EDUCATION: Record<AssetCategory, { title: string; description: string; driver: string }> = {
  Stocks: {
    title: "Individual Stocks",
    description: "Ownership shares in a single public company.",
    driver: "Driven by company earnings, future growth expectations, and overall economic health."
  },
  ETFs: {
    title: "Exchange Traded Funds",
    description: "A basket of many stocks or bonds, giving you instant diversification.",
    driver: "Driven by the collective performance of the underlying assets in the fund."
  },
  Crypto: {
    title: "Cryptocurrency",
    description: "Digital assets secured by cryptography, operating outside traditional banking.",
    driver: "Highly volatile; driven by adoption rates, regulatory news, and market sentiment."
  },
  RealEstate: {
    title: "Real Estate (REITs/Property)",
    description: "Investments in physical properties or trusts that manage real estate.",
    driver: "Driven by interest rates, housing supply/demand, and local economic growth."
  },
  Bonds: {
    title: "Bonds",
    description: "Essentially an I-OU. You lend money to a government or corporation for regular interest.",
    driver: "Prices move inversely to interest rates. When new rates go up, existing bond values go down."
  },
  PreciousMetals: {
    title: "Precious Metals",
    description: "Physical assets like Gold and Silver.",
    driver: "Often acts as a 'safe haven'. Driven by inflation fears and a weakening currency."
  },
  IntlEquities: {
    title: "International Equities",
    description: "Stocks from companies based outside your home country.",
    driver: "Driven by global trade, geopolitical events, and currency exchange rates."
  }
};
EOF

cat << 'EOF' > lib/api.ts
import { AssetCategory, NewsArticle, PriceData } from './types';

const MOCK_PRICES: Record<string, PriceData> = {
  'AAPL': { ticker: 'AAPL', currentPrice: 175.50, change24h: 1.2 },
  'VOO': { ticker: 'VOO', currentPrice: 480.20, change24h: 0.8 },
  'BTC': { ticker: 'BTC', currentPrice: 64200.00, change24h: -2.1 },
  'VNQ': { ticker: 'VNQ', currentPrice: 85.30, change24h: 0.4 },
  'BND': { ticker: 'BND', currentPrice: 72.10, change24h: 0.1 },
  'GLD': { ticker: 'GLD', currentPrice: 215.40, change24h: 0.5 },
  'VXUS': { ticker: 'VXUS', currentPrice: 60.80, change24h: -0.3 },
};

export async function fetchPrice(ticker: string, category: AssetCategory): Promise<PriceData> {
  const upperTicker = ticker.toUpperCase();
  const mockReturn = MOCK_PRICES[upperTicker] || { ticker: upperTicker, currentPrice: 100, change24h: 0 };

  if (category === 'Crypto') {
    try {
      const res = await fetch(`https://api.coingecko.com/api/v3/simple/price?ids=${ticker.toLowerCase()}&vs_currencies=usd&include_24hr_change=true`);
      const data = await res.json();
      const id = Object.keys(data)[0];
      if (id) {
        return {
          ticker: upperTicker,
          currentPrice: data[id].usd,
          change24h: data[id].usd_24h_change
        };
      }
    } catch (e) {
      console.warn("CoinGecko fetch failed, using fallback.");
    }
  }

  const alphaKey = process.env.NEXT_PUBLIC_ALPHA_VANTAGE_API_KEY;
  if (alphaKey && alphaKey !== 'your_alpha_vantage_key') {
    try {
      const res = await fetch(`https://www.alphavantage.co/query?function=GLOBAL_QUOTE&symbol=${upperTicker}&apikey=${alphaKey}`);
      const data = await res.json();
      if (data['Global Quote']) {
        return {
          ticker: upperTicker,
          currentPrice: parseFloat(data['Global Quote']['05. price']),
          change24h: parseFloat(data['Global Quote']['10. change percent'].replace('%', ''))
        };
      }
    } catch (e) {
      console.warn("Alpha Vantage fetch failed, using fallback.");
    }
  }

  return mockReturn;
}

const mapKeywordsToCategories = (text: string): AssetCategory[] => {
  const lower = text.toLowerCase();
  const categories = new Set<AssetCategory>();
  
  if (lower.includes('rate') || lower.includes('fed') || lower.includes('inflation')) categories.add('Bonds');
  if (lower.includes('crypto') || lower.includes('bitcoin') || lower.includes('sec')) categories.add('Crypto');
  if (lower.includes('housing') || lower.includes('mortgage') || lower.includes('real estate')) categories.add('RealEstate');
  if (lower.includes('gold') || lower.includes('silver')) categories.add('PreciousMetals');
  if (lower.includes('europe') || lower.includes('china') || lower.includes('global')) categories.add('IntlEquities');
  if (lower.includes('stock') || lower.includes('market') || lower.includes('earnings')) {
    categories.add('Stocks');
    categories.add('ETFs');
  }
  
  return Array.from(categories);
};

export async function fetchNews(): Promise<NewsArticle[]> {
  const newsKey = process.env.NEXT_PUBLIC_GNEWS_API_KEY;
  
  if (!newsKey || newsKey === 'your_gnews_key') {
    return [
      {
        title: "Federal Reserve Indicates Potential Rate Cuts Later This Year",
        description: "Inflation shows signs of cooling, prompting speculation that the central bank may lower borrowing costs.",
        url: "#",
        publishedAt: new Date().toISOString(),
        source: { name: "Financial Times Mock" },
        relatedCategories: ['Bonds', 'RealEstate', 'Stocks']
      },
      {
        title: "Bitcoin Surges Past Key Resistance Level Amid Regulatory Clarity",
        description: "Cryptocurrency markets rally as new global frameworks provide certainty for institutional investors.",
        url: "#",
        publishedAt: new Date().toISOString(),
        source: { name: "Crypto News Mock" },
        relatedCategories: ['Crypto']
      },
      {
        title: "Housing Market Inventory Remains Historically Low",
        description: "First-time buyers struggle as property owners hold onto low mortgage rates, restricting supply.",
        url: "#",
        publishedAt: new Date().toISOString(),
        source: { name: "Real Estate Journal Mock" },
        relatedCategories: ['RealEstate']
      }
    ];
  }

  try {
    const res = await fetch(`https://gnews.io/api/v4/top-headlines?category=business&lang=en&apikey=${newsKey}`);
    const data = await res.json();
    return data.articles.slice(0, 5).map((article: any) => ({
      ...article,
      relatedCategories: mapKeywordsToCategories(article.title + " " + article.description)
    }));
  } catch (e) {
    return [];
  }
}
EOF

cat << 'EOF' > lib/store.ts
import { create } from 'zustand';
import { Holding } from './types';

interface PortfolioState {
  holdings: Holding[];
  addHolding: (holding: Omit<Holding, 'id'>) => void;
  removeHolding: (id: string) => void;
}

export const usePortfolioStore = create<PortfolioState>((set) => ({
  holdings: [
    { id: '1', category: 'Stocks', ticker: 'AAPL', name: 'Apple Inc.', quantity: 10, purchasePrice: 150 },
    { id: '2', category: 'ETFs', ticker: 'VOO', name: 'Vanguard S&P 500', quantity: 5, purchasePrice: 400 },
    { id: '3', category: 'Bonds', ticker: 'BND', name: 'Total Bond Market', quantity: 20, purchasePrice: 70 },
  ],
  addHolding: (holding) => set((state) => ({
    holdings: [...state.holdings, { ...holding, id: Math.random().toString(36).substr(2, 9) }]
  })),
  removeHolding: (id) => set((state) => ({
    holdings: state.holdings.filter(h => h.id !== id)
  })),
}));
EOF

cat << 'EOF' > app/layout.tsx
import type { Metadata } from "next";
import { Inter } from "next/font/google";
import "./globals.css";

const inter = Inter({ subsets: ["latin"] });

export const metadata: Metadata = {
  title: "Lumina | Your Financial Companion",
  description: "Learn while you track your portfolio.",
};

export default function RootLayout({
  children,
}: Readonly<{
  children: React.ReactNode;
}>) {
  return (
    <html lang="en">
      <body className={inter.className}>
        <nav className="bg-white border-b border-slate-200 px-6 py-4 flex items-center justify-between shadow-sm">
          <div className="flex items-center gap-2">
            <div className="w-8 h-8 bg-brand-500 rounded-lg flex items-center justify-center text-white font-bold">L</div>
            <span className="text-xl font-semibold text-slate-800">Lumina Finance</span>
          </div>
          <div className="text-sm text-slate-500 font-medium">Educational Dashboard</div>
        </nav>
        <main className="max-w-7xl mx-auto p-4 sm:p-6 lg:p-8">
          {children}
        </main>
      </body>
    </html>
  );
}
EOF

cat << 'EOF' > components/EducationalCard.tsx
"use client";
import { BookOpen } from "lucide-react";
import { AssetCategory } from "@/lib/types";
import { CATEGORY_EDUCATION } from "@/lib/education";

export default function EducationalCard({ category }: { category: AssetCategory }) {
  const content = CATEGORY_EDUCATION[category];
  
  return (
    <div className="bg-brand-50 border border-brand-100 rounded-xl p-4 flex gap-4 items-start">
      <div className="bg-brand-100 p-2 rounded-lg text-brand-600 mt-1">
        <BookOpen size={20} />
      </div>
      <div>
        <h4 className="font-semibold text-brand-900">{content.title}</h4>
        <p className="text-sm text-brand-800 mt-1">{content.description}</p>
        <div className="mt-2 text-sm bg-white bg-opacity-60 p-2 rounded text-brand-900 italic">
          <span className="font-semibold">What drives value:</span> {content.driver}
        </div>
      </div>
    </div>
  );
}
EOF

cat << 'EOF' > components/PortfolioBuilder.tsx
"use client";
import { useState } from "react";
import { usePortfolioStore } from "@/lib/store";
import { AssetCategory } from "@/lib/types";
import EducationalCard from "./EducationalCard";
import { PlusCircle } from "lucide-react";

export default function PortfolioBuilder() {
  const addHolding = usePortfolioStore((state) => state.addHolding);
  const [category, setCategory] = useState<AssetCategory>("Stocks");
  const [ticker, setTicker] = useState("");
  const [name, setName] = useState("");
  const [quantity, setQuantity] = useState("");
  const [price, setPrice] = useState("");

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    if (!ticker || !quantity || !price) return;
    
    addHolding({
      category,
      ticker: ticker.toUpperCase(),
      name: name || ticker.toUpperCase(),
      quantity: Number(quantity),
      purchasePrice: Number(price),
    });
    
    setTicker(""); setName(""); setQuantity(""); setPrice("");
  };

  return (
    <div className="bg-white rounded-2xl shadow-sm border border-slate-200 p-6">
      <h2 className="text-lg font-bold text-slate-800 mb-4">Add New Asset</h2>
      <form onSubmit={handleSubmit} className="space-y-4">
        <div>
          <label className="block text-sm font-medium text-slate-600 mb-1">Asset Category</label>
          <select 
            value={category} 
            onChange={(e) => setCategory(e.target.value as AssetCategory)}
            className="w-full border border-slate-300 rounded-lg px-3 py-2 focus:ring-2 focus:ring-brand-500 focus:outline-none"
          >
            <option value="Stocks">Stocks</option>
            <option value="ETFs">ETFs</option>
            <option value="Crypto">Crypto (e.g., BTC, ETH)</option>
            <option value="RealEstate">Real Estate (REITs)</option>
            <option value="Bonds">Bonds</option>
            <option value="PreciousMetals">Precious Metals</option>
            <option value="IntlEquities">International Equities</option>
          </select>
        </div>

        <EducationalCard category={category} />

        <div className="grid grid-cols-2 gap-4">
          <div>
            <label className="block text-sm font-medium text-slate-600 mb-1">Ticker/Symbol</label>
            <input 
              required
              type="text" placeholder="e.g. AAPL" value={ticker} onChange={(e) => setTicker(e.target.value)}
              className="w-full border border-slate-300 rounded-lg px-3 py-2 focus:ring-2 focus:ring-brand-500"
            />
          </div>
          <div>
            <label className="block text-sm font-medium text-slate-600 mb-1">Friendly Name</label>
            <input 
              type="text" placeholder="Apple Inc." value={name} onChange={(e) => setName(e.target.value)}
              className="w-full border border-slate-300 rounded-lg px-3 py-2 focus:ring-2 focus:ring-brand-500"
            />
          </div>
          <div>
            <label className="block text-sm font-medium text-slate-600 mb-1">Quantity</label>
            <input 
              required
              type="number" step="any" min="0" placeholder="10" value={quantity} onChange={(e) => setQuantity(e.target.value)}
              className="w-full border border-slate-300 rounded-lg px-3 py-2 focus:ring-2 focus:ring-brand-500"
            />
          </div>
          <div>
            <label className="block text-sm font-medium text-slate-600 mb-1">Avg Buy Price ($)</label>
            <input 
              required
              type="number" step="any" min="0" placeholder="150.00" value={price} onChange={(e) => setPrice(e.target.value)}
              className="w-full border border-slate-300 rounded-lg px-3 py-2 focus:ring-2 focus:ring-brand-500"
            />
          </div>
        </div>

        <button type="submit" className="w-full bg-brand-600 hover:bg-brand-700 text-white font-medium py-2 rounded-lg flex justify-center items-center gap-2 transition-colors">
          <PlusCircle size={18} /> Add to Portfolio
        </button>
      </form>
    </div>
  );
}
EOF

cat << 'EOF' > components/PortfolioOverview.tsx
"use client";
import { useEffect, useState } from "react";
import { usePortfolioStore } from "@/lib/store";
import { fetchPrice } from "@/lib/api";
import { PriceData } from "@/lib/types";
import { Trash2, TrendingUp, TrendingDown, RefreshCcw } from "lucide-react";

export default function PortfolioOverview() {
  const { holdings, removeHolding } = usePortfolioStore();
  const [prices, setPrices] = useState<Record<string, PriceData>>({});
  const [loading, setLoading] = useState(true);

  const loadPrices = async () => {
    setLoading(true);
    const priceMap: Record<string, PriceData> = {};
    for (const h of holdings) {
      if (!priceMap[h.ticker]) {
        priceMap[h.ticker] = await fetchPrice(h.ticker, h.category);
      }
    }
    setPrices(priceMap);
    setLoading(false);
  };

  useEffect(() => {
    loadPrices();
  }, [holdings]);

  let totalInvested = 0;
  let totalCurrentValue = 0;

  const enrichedHoldings = holdings.map(h => {
    const currentPrice = prices[h.ticker]?.currentPrice || h.purchasePrice;
    const invested = h.quantity * h.purchasePrice;
    const current = h.quantity * currentPrice;
    const gainLoss = current - invested;
    const gainLossPct = (gainLoss / invested) * 100;
    
    totalInvested += invested;
    totalCurrentValue += current;

    return { ...h, currentPrice, current, gainLoss, gainLossPct };
  });

  const totalGainLoss = totalCurrentValue - totalInvested;
  const totalGainLossPct = totalInvested > 0 ? (totalGainLoss / totalInvested) * 100 : 0;

  return (
    <div className="bg-white rounded-2xl shadow-sm border border-slate-200 p-6 flex flex-col h-full">
      <div className="flex justify-between items-center mb-6">
        <h2 className="text-lg font-bold text-slate-800">Your Portfolio</h2>
        <button onClick={loadPrices} className="text-slate-400 hover:text-brand-600 transition-colors" title="Refresh Prices">
          <RefreshCcw size={18} className={loading ? "animate-spin" : ""} />
        </button>
      </div>

      <div className="bg-slate-50 rounded-xl p-4 mb-6 flex justify-between items-center border border-slate-100">
        <div>
          <p className="text-sm text-slate-500 font-medium">Total Value</p>
          <p className="text-3xl font-bold text-slate-800">
            ${totalCurrentValue.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 })}
          </p>
        </div>
        <div className={`text-right ${totalGainLoss >= 0 ? 'text-green-600' : 'text-red-600'}`}>
          <p className="text-sm font-medium text-slate-500 mb-1">All-Time Return</p>
          <div className="flex items-center justify-end gap-1 font-bold">
            {totalGainLoss >= 0 ? <TrendingUp size={20} /> : <TrendingDown size={20} />}
            ${Math.abs(totalGainLoss).toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 })} 
            <span className="text-sm font-medium ml-1">({totalGainLossPct.toFixed(2)}%)</span>
          </div>
        </div>
      </div>

      <div className="overflow-x-auto flex-1">
        {holdings.length === 0 ? (
          <div className="text-center text-slate-500 py-10">Your portfolio is empty. Add assets to begin.</div>
        ) : (
          <table className="w-full text-left border-collapse">
            <thead>
              <tr className="text-xs uppercase text-slate-400 border-b border-slate-100">
                <th className="pb-3 font-semibold">Asset</th>
                <th className="pb-3 font-semibold text-right">Qty</th>
                <th className="pb-3 font-semibold text-right">Avg Price</th>
                <th className="pb-3 font-semibold text-right">Current</th>
                <th className="pb-3 font-semibold text-right">Return</th>
                <th className="pb-3 font-semibold text-center">Action</th>
              </tr>
            </thead>
            <tbody>
              {enrichedHoldings.map((h) => (
                <tr key={h.id} className="border-b border-slate-50 hover:bg-slate-50 transition-colors">
                  <td className="py-3">
                    <div className="font-semibold text-slate-800">{h.ticker}</div>
                    <div className="text-xs text-slate-500">{h.name}</div>
                  </td>
                  <td className="py-3 text-right text-slate-700">{h.quantity}</td>
                  <td className="py-3 text-right text-slate-700">${h.purchasePrice.toFixed(2)}</td>
                  <td className="py-3 text-right font-medium text-slate-800">${h.currentPrice.toFixed(2)}</td>
                  <td className={`py-3 text-right font-medium ${h.gainLoss >= 0 ? 'text-green-600' : 'text-red-600'}`}>
                    {h.gainLoss >= 0 ? '+' : ''}{h.gainLossPct.toFixed(2)}%
                  </td>
                  <td className="py-3 text-center">
                    <button onClick={() => removeHolding(h.id)} className="text-slate-300 hover:text-red-500 transition-colors">
                      <Trash2 size={16} />
                    </button>
                  </td>
                </tr>
              ))}
            </tbody>
          </table>
        )}
      </div>
    </div>
  );
}
EOF

cat << 'EOF' > components/ForecastingTool.tsx
"use client";
import { useState, useMemo } from "react";
import { usePortfolioStore } from "@/lib/store";
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer } from 'recharts';
import { Calculator } from "lucide-react";

export default function ForecastingTool() {
  const holdings = usePortfolioStore(state => state.holdings);
  const currentTotal = holdings.reduce((sum, h) => sum + (h.quantity * h.purchasePrice), 0);

  const [years, setYears] = useState(10);
  const [monthlyContribution, setMonthlyContribution] = useState(500);
  const [expectedReturn, setExpectedReturn] = useState(7);
  const [inflationRate, setInflationRate] = useState(2.5);

  const forecastData = useMemo(() => {
    const data = [];
    const realReturnRate = (expectedReturn - inflationRate) / 100;
    const monthlyRate = realReturnRate / 12;
    
    let currentVal = currentTotal;
    let nominalVal = currentTotal;
    const currentYear = new Date().getFullYear();

    for (let i = 0; i <= years; i++) {
      data.push({
        year: currentYear + i,
        "Inflation-Adjusted Value": Math.round(currentVal),
        "Nominal Value": Math.round(nominalVal)
      });

      for(let m = 0; m < 12; m++) {
        currentVal = currentVal * (1 + monthlyRate) + monthlyContribution;
        nominalVal = nominalVal * (1 + (expectedReturn/100)/12) + monthlyContribution;
      }
    }
    return data;
  }, [currentTotal, years, monthlyContribution, expectedReturn, inflationRate]);

  return (
    <div className="bg-white rounded-2xl shadow-sm border border-slate-200 p-6">
      <div className="flex items-center gap-2 mb-6">
        <Calculator className="text-brand-500" />
        <h2 className="text-lg font-bold text-slate-800">Growth Forecaster</h2>
      </div>

      <div className="grid grid-cols-1 md:grid-cols-4 gap-4 mb-8">
        <div>
          <label className="block text-sm font-medium text-slate-600 mb-1">Time Horizon (Years): {years}</label>
          <input type="range" min="1" max="40" value={years} onChange={(e) => setYears(Number(e.target.value))} className="w-full accent-brand-500" />
        </div>
        <div>
          <label className="block text-sm font-medium text-slate-600 mb-1">Monthly Add ($): {monthlyContribution}</label>
          <input type="range" min="0" max="5000" step="50" value={monthlyContribution} onChange={(e) => setMonthlyContribution(Number(e.target.value))} className="w-full accent-brand-500" />
        </div>
        <div>
          <label className="block text-sm font-medium text-slate-600 mb-1">Expected Return: {expectedReturn}%</label>
          <input type="range" min="1" max="15" step="0.5" value={expectedReturn} onChange={(e) => setExpectedReturn(Number(e.target.value))} className="w-full accent-brand-500" />
        </div>
        <div>
          <label className="block text-sm font-medium text-slate-600 mb-1">Inflation Rate: {inflationRate}%</label>
          <input type="range" min="0" max="10" step="0.5" value={inflationRate} onChange={(e) => setInflationRate(Number(e.target.value))} className="w-full accent-brand-500" />
        </div>
      </div>

      <div className="bg-slate-50 border border-slate-100 rounded-xl p-4 mb-6">
        <p className="text-sm text-slate-600">
          <strong>How this works:</strong> We use <em>Compound Growth</em>. By earning a return on your past returns, money snowballs over time. The chart shows your real buying power (adjusted for inflation) vs the nominal dollar amount.
        </p>
      </div>

      <div className="h-72 w-full">
        <ResponsiveContainer width="100%" height="100%">
          <LineChart data={forecastData} margin={{ top: 5, right: 20, bottom: 5, left: 0 }}>
            <CartesianGrid strokeDasharray="3 3" vertical={false} stroke="#e2e8f0" />
            <XAxis dataKey="year" stroke="#94a3b8" fontSize={12} tickLine={false} axisLine={false} />
            <YAxis stroke="#94a3b8" fontSize={12} tickLine={false} axisLine={false} tickFormatter={(value) => `$${value >= 1000 ? (value/1000).toFixed(0) + 'k' : value}`} />
            <Tooltip formatter={(value: number) => `$${value.toLocaleString()}`} />
            <Line type="monotone" dataKey="Inflation-Adjusted Value" stroke="#3b82f6" strokeWidth={3} dot={false} />
            <Line type="monotone" dataKey="Nominal Value" stroke="#94a3b8" strokeWidth={2} strokeDasharray="5 5" dot={false} />
          </LineChart>
        </ResponsiveContainer>
      </div>
    </div>
  );
}
EOF

cat << 'EOF' > components/NewsFeed.tsx
"use client";
import { useEffect, useState } from "react";
import { fetchNews } from "@/lib/api";
import { NewsArticle } from "@/lib/types";
import { usePortfolioStore } from "@/lib/store";
import { Newspaper, ExternalLink } from "lucide-react";
import { formatDistanceToNow } from "date-fns";

export default function NewsFeed() {
  const [news, setNews] = useState<NewsArticle[]>([]);
  const holdings = usePortfolioStore(state => state.holdings);
  const userCategories = new Set(holdings.map(h => h.category));

  useEffect(() => {
    fetchNews().then(setNews);
  }, []);

  return (
    <div className="bg-white rounded-2xl shadow-sm border border-slate-200 p-6">
      <div className="flex items-center gap-2 mb-6">
        <Newspaper className="text-brand-500" />
        <h2 className="text-lg font-bold text-slate-800">World Events & Impact</h2>
      </div>

      <div className="space-y-4">
        {news.length === 0 ? (
          <p className="text-slate-500 text-sm">Loading market news...</p>
        ) : (
          news.map((article, i) => {
            const impactCategories = article.relatedCategories.filter(c => userCategories.has(c));
            
            return (
              <div key={i} className="border-b border-slate-100 last:border-0 pb-4 last:pb-0">
                <a href={article.url} target="_blank" rel="noreferrer" className="group">
                  <h3 className="font-semibold text-slate-800 group-hover:text-brand-600 transition-colors flex items-start justify-between">
                    <span>{article.title}</span>
                    <ExternalLink size={14} className="opacity-0 group-hover:opacity-100 mt-1 flex-shrink-0" />
                  </h3>
                  <p className="text-sm text-slate-500 mt-1 line-clamp-2">{article.description}</p>
                </a>
                
                <div className="flex items-center justify-between mt-3">
                  <span className="text-xs text-slate-400">
                    {article.source.name} • {formatDistanceToNow(new Date(article.publishedAt))} ago
                  </span>
                  
                  {impactCategories.length > 0 && (
                    <div className="flex gap-1">
                      {impactCategories.map(cat => (
                        <span key={cat} className="text-[10px] uppercase font-bold bg-amber-100 text-amber-800 px-2 py-0.5 rounded-full" title={`May impact your ${cat} holdings`}>
                          Impacts {cat}
                        </span>
                      ))}
                    </div>
                  )}
                </div>
              </div>
            );
          })
        )}
      </div>
      <div className="mt-4 pt-4 border-t border-slate-100">
        <p className="text-xs text-slate-400 italic">News categories are mapped based on keywords. If a category is highlighted in yellow, you own assets that might be affected by this event.</p>
      </div>
    </div>
  );
}
EOF

cat << 'EOF' > app/page.tsx
import PortfolioBuilder from "@/components/PortfolioBuilder";
import PortfolioOverview from "@/components/PortfolioOverview";
import ForecastingTool from "@/components/ForecastingTool";
import NewsFeed from "@/components/NewsFeed";

export default function Home() {
  return (
    <div className="space-y-6">
      <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
        <div className="lg:col-span-2">
          <PortfolioOverview />
        </div>
        <div className="lg:col-span-1">
          <PortfolioBuilder />
        </div>
      </div>
      <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
        <div className="lg:col-span-2">
          <ForecastingTool />
        </div>
        <div className="lg:col-span-1">
          <NewsFeed />
        </div>
      </div>
    </div>
  );
}
EOF

# 5. Start the development server
echo "Setup complete! Starting the Next.js development server..."
npm run dev