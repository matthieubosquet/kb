# syntax=docker/dockerfile:1

ARG DOTNET_VERSION=10.0
ARG NAMEDQUERY_VERSION=0.1.0-alpha.2


## ---- Stage: default build context value for the endpoint definitions
FROM scratch AS endpoint-definitions
COPY --exclude=Dockerfile . /


## ---- Stage: build a minimal host app around the TheODI.NamedQuery package
FROM mcr.microsoft.com/dotnet/sdk:${DOTNET_VERSION} AS build

ARG NAMEDQUERY_VERSION

WORKDIR /src

COPY <<EOF App.csproj
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <InvariantGlobalization>true</InvariantGlobalization>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="TheODI.NamedQuery" Version="${NAMEDQUERY_VERSION}" />
  </ItemGroup>
</Project>
EOF

COPY <<EOF Program.cs
using Microsoft.Extensions.FileProviders;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddNamedQuery(options =>
    options.Definitions = new PhysicalFileProvider(Path.Combine(AppContext.BaseDirectory, "endpoint-definitions")));

var app = builder.Build();
app.UseNamedQuery();

app.Run();
EOF

RUN dotnet restore
RUN dotnet publish --no-restore -c Release -o /app


## ---- Stage: runtime
FROM mcr.microsoft.com/dotnet/aspnet:${DOTNET_VERSION}-noble-chiseled

WORKDIR /app
COPY --link --from=build /app .
COPY --link --from=endpoint-definitions . ./endpoint-definitions/

EXPOSE 8080

ENTRYPOINT ["dotnet", "App.dll"]
